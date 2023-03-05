# Chain_Indexer

​	    之前跟着网上区块链大佬看以太坊源码，学到了很多东西。但是大佬有些地方讲得不详细，对于golang和eth的双初学者不够友好，于是就有了自己写源码分析的念头。目前跟着大佬学习到了core/chain_Indexer。</br>

​	    chain_Indexer实际上是维护bloom工作，对section中每个区块头的logsbloom整合成一个大的bloom放在ChainIndexer.backend里面。有新的区块产生，则产生新的bloom。

### NewChainIndexer

c := &ChainIndexer{...}就是非常常规的初始化，c.loadValidSections()是有可能这个indexDb中原来就成功索引到数据库的部分数量，那么就进行一个加载，c.updateLoop()是一个保持更新的作用。

```go
func NewChainIndexer(chainDb, indexDb ethdb.Database, backend ChainIndexerBackend, section, confirm uint64, throttling time.Duration, kind string) *ChainIndexer {
   c := &ChainIndexer{
      chainDb:     chainDb,
      indexDb:     indexDb,
      backend:     backend,
      update:      make(chan struct{}, 1),
      quit:        make(chan chan error),
      sectionSize: section,
      confirmsReq: confirm,
      throttling:  throttling,
      log:         log.New("type", kind),
   }
   // Initialize database dependent fields and start the updater
   c.loadValidSections()
   go c.updateLoop()
   return c
}
```

c.loadValidSections()：从indexDb中取出关键词"count"的数据，然后检验合法性，最后存储到storedSections中

```golang
func (c *ChainIndexer) loadValidSections() {
   //count就是数量
   data, _ := c.indexDb.Get([]byte("count"))
   if len(data) == 8 {
      //如果数量合法，那么就存到c.storedSections
      c.storedSections = binary.BigEndian.Uint64(data[:])
   }
}
```

c.updateLoop():进入一个while循环。1.退出条件，管道c.quit收到退出信息，忽略err,结束Loop。2.获得更新通知，如果已知的section>已经更新过的section，那么进行更新。更新主要就是对一个sectionSize(通常4096)里的区块，将他们的head里的bloom复制到c.backend.gen.blooms里，主要过程是在newHead, err := c.processSection(section, oldHead)。同时对于他们孩子区块提示他们更新。最后如果没有结束，则再开展更新操作。

```go
func (c *ChainIndexer) updateLoop() {
	var (
		updating bool
		updated  time.Time
	)
	//while循环
	for {
		select {
		case errc := <-c.quit:
			// Chain indexer terminating, report no failure and abort
			errc <- nil
			return

		case <-c.update:
			// Section headers completed (or rolled back), update the index
			c.lock.Lock()
			if c.knownSections > c.storedSections {
				// Periodically print an upgrade log message to the user
				// 如果超出一定时间&已经检索到的c>存储的c，那么记录日志
				if time.Since(updated) > 8*time.Second {
					if c.knownSections > c.storedSections+1 {
						updating = true
						c.log.Info("Upgrading chain index", "percentage", c.storedSections*100/c.knownSections)
					}
					updated = time.Now()
				}
				// Cache the current section count and head to allow unlocking the mutex
				section := c.storedSections
				var oldHead common.Hash
				if section > 0 {
					//根据section从数据库里拿Head
					oldHead = c.SectionHead(section - 1)
				}
				// Process the newly defined section in the background
				c.lock.Unlock()
				//将bloom从head里复制到c.backend.gen.blooms里
				newHead, err := c.processSection(section, oldHead)
				if err != nil {
					c.log.Error("Section processing failed", "error", err)
				}
				c.lock.Lock()

				// If processing succeeded and no reorgs occcurred, mark the section completed
				if err == nil && oldHead == c.SectionHead(section-1) {
					//将section和对应的newHead存入数据库
					c.setSectionHead(section, newHead)
					c.setValidSections(section + 1)
					if c.storedSections == c.knownSections && updating {
						updating = false
						c.log.Info("Finished upgrading chain index")
					}
					//cascadedHead 是更新后的section的最后一个区块的高度
					c.cascadedHead = c.storedSections*c.sectionSize - 1
					for _, child := range c.children {
						c.log.Trace("Cascading chain index update", "head", c.cascadedHead)
						//孩子也进行更新到最新的区块
						child.newHead(c.cascadedHead, false)
					}
				} else {
					// If processing failed, don't retry until further notification
					c.log.Debug("Chain index processing failed", "section", section, "err", err)
					c.knownSections = c.storedSections
				}
			}
			// If there are still further sections to process, reschedule
			// 如果更新尚未成功，那么继续更新！！！
			if c.knownSections > c.storedSections {
				time.AfterFunc(c.throttling, func() {
					select {
					case c.update <- struct{}{}:
					default:
					}
				})
			}
			c.lock.Unlock()
		}
	}
}

```

c.setValidSections()：更新数据库indexDb中存储的count，也就是审计过的数量，同时删除存储的所有大于section的头哈希

```go
func (c *ChainIndexer) setValidSections(sections uint64) {
   // Set the current number of valid sections in the database
   var data [8]byte
   binary.BigEndian.PutUint64(data[:], sections)
   c.indexDb.Put([]byte("count"), data[:])

   // Remove any reorged sections, caching the valids in the mean time
   for c.storedSections > sections {
      c.storedSections--
      c.removeSectionHead(c.storedSections)
   }
   c.storedSections = sections // needed if new > old
}
```

processSection()：对这个section里的所有block增加logsbloom并存入数据库。

```go
func (c *ChainIndexer) processSection(section uint64, lastHead common.Hash) (common.Hash, error) {
   c.log.Trace("Processing new chain section", "section", section)

   // Reset and partial processing
   // reset就是新建位置是section的bloom,启动这个新的bloom
   if err := c.backend.Reset(section, lastHead); err != nil {
      c.setValidSections(0)
      return common.Hash{}, err
   }
   //对这一个section里的区块进行
   for number := section * c.sectionSize; number < (section+1)*c.sectionSize; number++ {
      //获得对应区块的哈希<-数据库
      hash := GetCanonicalHash(c.chainDb, number)
      if hash == (common.Hash{}) {
         return common.Hash{}, fmt.Errorf("canonical block #%d unknown", number)
      }
      //获得区块头<-数据库
      header := GetHeader(c.chainDb, hash, number)
      if header == nil {
         return common.Hash{}, fmt.Errorf("block #%d [%x…] not found", number, hash[:4])
      } else if header.ParentHash != lastHead {
         return common.Hash{}, fmt.Errorf("chain reorged during section processing")
      }
      //获得这个header的logsbloom
      c.backend.Process(header)
      lastHead = header.Hash()
   }
    //logsbloom->数据库
   if err := c.backend.Commit(); err != nil {
      c.log.Error("Section commit failed", "error", err)
      return common.Hash{}, err
   }
   return lastHead, nil
}
```

c.backend.Process(header)跳转到的AddBloom()代码解析如下，其实就是对于每一个区块，计算在Generator中的位置(在哪一个字节byteIndex，哪一位bitMask)，然后对这个区块的bloom的每一位i进行分析，如果为1就把对应Generator的位置置1。就是一个复制的过程。

```go
func (b *Generator) AddBloom(index uint, bloom types.Bloom) error {
   // Make sure we're not adding more bloom filters than our capacity
   if b.nextBit >= b.sections {
      return errSectionOutOfBounds
   }
   if b.nextBit != index {
      return errors.New("bloom filter with unexpected index")
   }
   // Rotate the bloom and insert into our collection
   //nextBit未被赋值，一开始是0，其实就是这个block所在的位置，(因为一列是8个字节，而block只占一位)
   byteIndex := b.nextBit / 8                //查找到对应的byte，需要设置这个byte位置
   bitMask := byte(1) << byte(7-b.nextBit%8) // 找到需要设置值的bit在byte的下标

   for i := 0; i < types.BloomBitLength; i++ {
      //
      bloomByteIndex := types.BloomByteLength - 1 - i/8 //从后往前数第几个byte
      bloomBitMask := byte(1) << byte(i%8)              //byte中的第几位

      if (bloom[bloomByteIndex] & bloomBitMask) != 0 { //如果bloom中的这位是1
         b.blooms[i][byteIndex] |= bitMask //将blooms的赋值为1
      }
   }
   b.nextBit++

   return nil
}
```

newHead主要就是起到一个更新作用，有两种更新方式:1.重组更新，重组更新就是将head之后的所有已经存储的section全部无效化。2.普通更新，就是已知head已经超过了存储在数据库中的section，那么就给c.update发送消息。

```go
func (c *ChainIndexer) newHead(head uint64, reorg bool) {
   c.lock.Lock()
   defer c.lock.Unlock()

   // If a reorg happened, invalidate all sections until that point
   //如果发生重组，将该点之后的所有部分无效
   if reorg {
      // Revert the known section number to the reorg point
      changed := head / c.sectionSize
      if changed < c.knownSections {
         c.knownSections = changed
      }
      // Revert the stored sections from the database to the reorg point
      if changed < c.storedSections {
         c.setValidSections(changed)
      }
      // Update the new head number to te finalized section end and notify children
      head = changed * c.sectionSize
      //ascadedHead 是更新后的section的最后一个区块的高度
      if head < c.cascadedHead {
         c.cascadedHead = head
         for _, child := range c.children {
            child.newHead(c.cascadedHead, true)
         }
      }
      return
   }
   // No reorg, calculate the number of newly known sections and update if high enough
   var sections uint64
   if head >= c.confirmsReq {
      sections = (head + 1 - c.confirmsReq) / c.sectionSize
      if sections > c.knownSections {
         c.knownSections = sections

         select {
         case c.update <- struct{}{}:
         default:
         }
      }
   }
}
```

### START

Start()起到一个监听信息的作用并通知bloom进行更新，events是一个管道，这个管道是比较有特点的，可以接收到广播信息，关于event.feed的详细可以参考https://pkg.go.dev/github.com/ethereum/go-ethereum/event。

```go
// Start 创建一个 goroutine 将链头事件馈送到索引器中进行级联后台处理。孩子不需要启动，他们的父母会通知他们有关新事件的信息。
func (c *ChainIndexer) Start(chain ChainIndexerChain) {
   events := make(chan ChainEvent, 10)
   sub := chain.SubscribeChainEvent(events)

   go c.eventLoop(chain.CurrentHeader(), events, sub)
}
```

eventLoop()：如果接收到了消息，分析出消息中含有的区块头的部分，进行bloom的头的更新

```go
func (c *ChainIndexer) eventLoop(currentHeader *types.Header, events chan ChainEvent, sub event.Subscription) {
   // Mark the chain indexer as active, requiring an additional teardown
   atomic.StoreUint32(&c.active, 1)

   defer sub.Unsubscribe()

   // Fire the initial new head event to start any outstanding processing
   c.newHead(currentHeader.Number.Uint64(), false)

   var (
      prevHeader = currentHeader
      prevHash   = currentHeader.Hash()
   )
   for {
      select {
      case errc := <-c.quit:
         // Chain indexer terminating, report no failure and abort
         errc <- nil
         return

      case ev, ok := <-events:
         // Received a new event, ensure it's not nil (closing) and update
         if !ok {
            errc := <-c.quit
            errc <- nil
            return
         }
         header := ev.Block.Header()
         if header.ParentHash != prevHash {
            // Reorg to the common ancestor (might not exist in light sync mode, skip reorg then)
            // TODO(karalabe, zsfelfoldi): This seems a bit brittle, can we detect this case explicitly?
            //如果出现了分叉，那么我们首先找到公共祖先， 从公共祖先之后的索引需要重建。
            if h := FindCommonAncestor(c.chainDb, prevHeader, header); h != nil {
               c.newHead(h.Number.Uint64(), true)
            }
         }
         //调用newHead进行更新
         c.newHead(header.Number.Uint64(), false)

         prevHeader, prevHash = header, header.Hash()
      }
   }
}
```

跟着大佬的节奏，到此Chain_Indexer的内容基本涵盖。