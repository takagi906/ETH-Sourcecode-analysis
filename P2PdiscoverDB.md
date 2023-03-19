# P2P/discover/database

首先是数据结构，经过布隆过滤器的大洗礼发现，数据结构是最重要的

```go
type nodeDB struct {
   lvl *leveldb.DB // Interface to the database itself
   //防止节点自身信息存储在数据库里，防止A->A发送握手请求循环
   self NodeID // Own node id to prevent adding it into the database
   //用来检查过期节点的线程
   runner sync.Once     // Ensures we can start at most one expirer
   quit   chan struct{} // Channel to signal the expiring thread to stop
}
```

了解了buffer的作用。通常用作处理字节数据的灵活高效的方式，特别是在输入或输出数据的大小事先未知的情况下特别有用。

建立新的nodeDB的两种方式，基于内存，基于数据库

```go
// path是给定的路径创建数据库的路径
func newNodeDB(path string, version int, self NodeID) (*nodeDB, error) {
   if path == "" {
      //如果给定的path是空的，则建立基于内存的
      return newMemoryNodeDB(self)
   }
   //建立一个持久化的节点DB
   return newPersistentNodeDB(path, version, self)
}

// newMemoryNodeDB creates a new in-memory node database without a persistent
// backend.
func newMemoryNodeDB(self NodeID) (*nodeDB, error) {
   //建立了基于内存的数据库
   db, err := leveldb.Open(storage.NewMemStorage(), nil)
   if err != nil {
      return nil, err
   }
   return &nodeDB{
      lvl:  db,
      self: self,
      quit: make(chan struct{}),
   }, nil
}

// newPersistentNodeDB creates/opens a leveldb backed persistent node database,
// also flushing its contents in case of a version mismatch.
func newPersistentNodeDB(path string, version int, self NodeID) (*nodeDB, error) {
   //设置缓存的最大文件数
   opts := &opt.Options{OpenFilesCacheCapacity: 5}
   db, err := leveldb.OpenFile(path, opts)
   if _, iscorrupted := err.(*errors.ErrCorrupted); iscorrupted {
      //如果数据存储已经损坏
      db, err = leveldb.RecoverFile(path, nil)
   }
   if err != nil {
      return nil, err
   }
   // The nodes contained in the cache correspond to a certain protocol version.
   // Flush all nodes if the version doesn't match.
   //current是现在的版本，version->current中
   currentVer := make([]byte, binary.MaxVarintLen64)
   currentVer = currentVer[:binary.PutVarint(currentVer, int64(version))]

   blob, err := db.Get(nodeDBVersionKey, nil)
   switch err {
   case leveldb.ErrNotFound:
      // Version not found (i.e. empty cache), insert it
      if err := db.Put(nodeDBVersionKey, currentVer, nil); err != nil {
         db.Close()
         return nil, err
      }

   case nil:
      // Version present, flush if different

      if !bytes.Equal(blob, currentVer) {
         //如果版本不同。可能存在数据差异，将缓冲区中的数据强制刷新到存储设备中
         db.Close()
         //把path路径下的所有文件夹都给删除，建立新的文件夹，这样子下一次blob应该就差不多东西了
         if err = os.RemoveAll(path); err != nil {
            return nil, err
         }
         return newPersistentNodeDB(path, version, self)
      }
   }
   return &nodeDB{
      lvl:  db,
      self: self,
      quit: make(chan struct{}),
   }, nil
}
```

利用id和field建立一把查询钥匙以及分解查询钥匙

```go
// makeKey generates the leveldb key-blob from a node id and its particular
// field of interest.
// 将节点 ID 和特定的字段转换为 LevelDB 数据库中的 key-blob
func makeKey(id NodeID, field string) []byte {
   if bytes.Equal(id[:], nodeDBNilNodeID[:]) {
      return []byte(field)
   }
   return append(nodeDBItemPrefix, append(id[:], field...)...)
}

// splitKey tries to split a database key into a node id and a field part.
// 将id与field进行拆开
func splitKey(key []byte) (id NodeID, field string) {
   // If the key is not of a node, return it plainly
   if !bytes.HasPrefix(key, nodeDBItemPrefix) {
      return NodeID{}, string(key)
   }
   // Otherwise split the id and field
   item := key[len(nodeDBItemPrefix):]
   copy(id[:], item[:len(id)])
   field = string(item[len(id):])

   return id, field
}

// fetchInt64 retrieves an integer instance associated with a particular
// database key.
// 拿key从db中取数，将数转换为10进制
func (db *nodeDB) fetchInt64(key []byte) int64 {
   blob, err := db.lvl.Get(key, nil)
   if err != nil {
      return 0
   }
   val, read := binary.Varint(blob)
   if read <= 0 {
      return 0
   }
   return val
}

// storeInt64 update a specific database entry to the current time instance as a
// unix timestamp.
// 将int64类型的数转换为字节数组类型，并存入数据库
func (db *nodeDB) storeInt64(key []byte, n int64) error {
   blob := make([]byte, binary.MaxVarintLen64)
   blob = blob[:binary.PutVarint(blob, n)]

   return db.lvl.Put(key, blob, nil)
}

// node retrieves a node with a given id from the database.
func (db *nodeDB) node(id NodeID) *Node {
   //这里的blob指的是RLP编码的node,关于P2P的Node
   blob, err := db.lvl.Get(makeKey(id, nodeDBDiscoverRoot), nil)
   if err != nil {
      return nil
   }
   node := new(Node)
   if err := rlp.DecodeBytes(blob, node); err != nil {
      log.Error("Failed to decode node RLP", "err", err)
      return nil
   }
   //sha是节点距离计算中使用的缓存副本
   node.sha = crypto.Keccak256Hash(node.ID[:])
   return node
}

// updateNode inserts - potentially overwriting - a node into the peer database.
// 将更新的节点写入db
func (db *nodeDB) updateNode(node *Node) error {
   blob, err := rlp.EncodeToBytes(node)
   if err != nil {
      return err
   }
   return db.lvl.Put(makeKey(node.ID, nodeDBDiscoverRoot), blob, nil)
}

// deleteNode deletes all information/keys associated with a node.
// 清除所有关于这个node的信息以及key
func (db *nodeDB) deleteNode(id NodeID) error {
   //生成一个具有id前缀的deleter
   deleter := db.lvl.NewIterator(util.BytesPrefix(makeKey(id, "")), nil)
   for deleter.Next() {
      //删除所有有关这个id的信息
      if err := db.lvl.Delete(deleter.Key(), nil); err != nil {
         return err
      }
   }
   return nil
}

// ensureExpirer is a small helper method ensuring that the data expiration
// mechanism is running. If the expiration goroutine is already running, this
// method simply returns.
//
// The goal is to start the data evacuation only after the network successfully
// bootstrapped itself (to prevent dumping potentially useful seed nodes). Since
// it would require significant overhead to exactly trace the first successful
// convergence, it's simpler to "ensure" the correct state when an appropriate
// condition occurs (i.e. a successful bonding), and discard further events.

```

以及确认是否过期的信息

```go
func (db *nodeDB) ensureExpirer() {
   db.runner.Do(func() { go db.expirer() })
}

// expirer should be started in a go routine, and is responsible for looping ad
// infinitum and dropping stale data from the database.
func (db *nodeDB) expirer() {
   // 定时器机制
   tick := time.Tick(nodeDBCleanupCycle)
   for {
      select {
      case <-tick:
         if err := db.expireNodes(); err != nil {
            log.Error("Failed to expire nodedb items", "err", err)
         }

      case <-db.quit:
         return
      }
   }
}

// expireNodes iterates over the database and deletes all nodes that have not
// been seen (i.e. received a pong from) for some allotted time.
func (db *nodeDB) expireNodes() error {
   //指阈值 超过一天没有更新
   threshold := time.Now().Add(-nodeDBNodeExpiration)

   // Find discovered nodes that are older than the allowance
   it := db.lvl.NewIterator(nil, nil)
   defer it.Release()

   for it.Next() {
      // Skip the item if not a discovery node
      id, field := splitKey(it.Key())
      if field != nodeDBDiscoverRoot {
         continue
      }
      // Skip the node if not expired yet (and not self)
      if !bytes.Equal(id[:], db.self[:]) {
         //如果这个不是自身节点，db.self是自身节点
         if seen := db.lastPong(id); seen.After(threshold) {
            continue
         }
      }
      // Otherwise delete all associated information
      db.deleteNode(id)
   }
   return nil
}
```

最重要的寻找节点

```go
// 如何找到其他节点并建立连接
func (db *nodeDB) querySeeds(n int, maxAge time.Duration) []*Node {
   var (
      now   = time.Now()
      nodes = make([]*Node, 0, n)
      it    = db.lvl.NewIterator(nil, nil)
      id    NodeID
   )
   defer it.Release()

seek:
   for seeks := 0; len(nodes) < n && seeks < n*5; seeks++ {
      // Seek to a random entry. The first byte is incremented by a
      // random amount each time in order to increase the likelihood
      // of hitting all existing nodes in very small databases.
      //随机生成一个id,寻找一个随机条目。 第一个字节每次增加一个随机数量，以增加命中非常小的数据库中所有现有节点的可能性。
      ctr := id[0]
      rand.Read(id[:])
      id[0] = ctr + id[0]%16
      //寻找关于这个id+key的所有节点
      it.Seek(makeKey(id, nodeDBDiscoverRoot))

      n := nextNode(it)
      if n == nil {
         id[0] = 0
         continue seek // iterator exhausted
      }
      if n.ID == db.self {
         continue seek
      }
      if now.Sub(db.lastPong(n.ID)) > maxAge {
         continue seek
      }
      for i := range nodes {
         if nodes[i].ID == n.ID {
            continue seek // duplicate
         }
      }
      nodes = append(nodes, n)
   }
   return nodes
}

// reads the next node record from the iterator, skipping over other
// database entries.
func nextNode(it iterator.Iterator) *Node {
   for end := false; !end; end = !it.Next() {
      id, field := splitKey(it.Key())
      if field != nodeDBDiscoverRoot {
         continue
      }
      var n Node
      if err := rlp.DecodeBytes(it.Value(), &n); err != nil {
         log.Warn("Failed to decode node RLP", "id", id, "err", err)
         continue
      }
      return &n
   }
   return nil
}
```