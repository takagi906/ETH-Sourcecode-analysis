# account_cache

在内存中存储一些account而不是硬盘中存储，用于加快读取速度

```go
// 两次缓存重新加载间隔的最短时间。 如果平台不支持更改通知，则此限制适用。
// 如果密钥库目录尚不存在，它也适用，代码将最多尝试创建一个观察者。
// 限制为2s
const minReloadInterval = 2 * time.Second

// Account集合
type accountsByURL []accounts.Account

// 按照URL排序的acounts
func (s accountsByURL) Len() int           { return len(s) }
func (s accountsByURL) Less(i, j int) bool { return s[i].URL.Cmp(s[j].URL) < 0 }
func (s accountsByURL) Swap(i, j int)      { s[i], s[j] = s[j], s[i] }

// AmbiguousAddrError is returned when attempting to unlock
// an address for which more than one file exists.
// 尝试解锁存在多个文件的地址时报错的数据结构
type AmbiguousAddrError struct {
   Addr    common.Address
   Matches []accounts.Account
}

// 报错指的是将一个地址对应的多个账户都打印出来，证明解锁失败
func (err *AmbiguousAddrError) Error() string {
   files := ""
   for i, a := range err.Matches {
      files += a.URL.Path
      if i < len(err.Matches)-1 {
         files += ", "
      }
   }
   return fmt.Sprintf("multiple keys match address (%s)", files)
}

// accountCache is a live index of all accounts in the keystore.
// accountCache 是密钥库中所有帐户的实时索引。
type accountCache struct {
   keydir string
   //目录监视是一种在计算机操作系统中常见的技术，它用于监视一个或多个目录中的文件或子目录的变化，例如文件的创建、修改、删除或重命名。
   //当目录中的文件或子目录发生变化时，监视程序会自动检测到并触发相应的事件或操作
   watcher *watcher
   mu      sync.Mutex
   all     accountsByURL
   byAddr  map[common.Address][]accounts.Account
   //函数节流可以用于限制函数的调用频率，从而降低函数被频繁调用时的资源消耗，提高程序的性能和响应速度。
   //通常情况下，函数节流会延迟函数的执行，直到一定时间内没有再次调用该函数时才会执行，从而避免函数被过度调用。
   throttle *time.Timer
   notify   chan struct{}
   fileC    fileCache
}

func newAccountCache(keydir string) (*accountCache, chan struct{}) {
   ac := &accountCache{
      keydir: keydir,
      byAddr: make(map[common.Address][]accounts.Account),
      notify: make(chan struct{}, 1),
      fileC:  fileCache{all: set.NewNonTS()},
   }
   ac.watcher = newWatcher(ac)
   return ac, ac.notify
}

// 对accountCache读的时候需要上锁，最后关锁
// 返回accountCache中的账号，无法修改
func (ac *accountCache) accounts() []accounts.Account {
   ac.maybeReload()
   ac.mu.Lock()
   defer ac.mu.Unlock()
   cpy := make([]accounts.Account, len(ac.all))
   copy(cpy, ac.all)
   return cpy
}

// 判断是否有这个地址的账户
func (ac *accountCache) hasAddress(addr common.Address) bool {
   ac.maybeReload()
   ac.mu.Lock()
   defer ac.mu.Unlock()
   return len(ac.byAddr[addr]) > 0
}

// 插入需要关锁，找到应该存放的下标，如果存在立刻返回
// 不存在就append空，所有往后移一位，插入，更新byAddr
func (ac *accountCache) add(newAccount accounts.Account) {
   ac.mu.Lock()
   defer ac.mu.Unlock()

   i := sort.Search(len(ac.all), func(i int) bool { return ac.all[i].URL.Cmp(newAccount.URL) >= 0 })
   if i < len(ac.all) && ac.all[i] == newAccount {
      return
   }
   // newAccount is not in the cache.
   ac.all = append(ac.all, accounts.Account{})
   copy(ac.all[i+1:], ac.all[i:])
   ac.all[i] = newAccount
   ac.byAddr[newAccount.Address] = append(ac.byAddr[newAccount.Address], newAccount)
}

// note: removed needs to be unique here (i.e. both File and Address must be set).
func (ac *accountCache) delete(removed accounts.Account) {
   ac.mu.Lock()
   defer ac.mu.Unlock()

   ac.all = removeAccount(ac.all, removed)
   if ba := removeAccount(ac.byAddr[removed.Address], removed); len(ba) == 0 {
      delete(ac.byAddr, removed.Address)
   } else {
      ac.byAddr[removed.Address] = ba
   }
}

// deleteByFile removes an account referenced by the given path.
func (ac *accountCache) deleteByFile(path string) {
   ac.mu.Lock()
   defer ac.mu.Unlock()
   i := sort.Search(len(ac.all), func(i int) bool { return ac.all[i].URL.Path >= path })

   if i < len(ac.all) && ac.all[i].URL.Path == path {
      removed := ac.all[i]
      //删除i这个下标，这里append的意思是ac.all[i+1]之后的元素一个一个加进，唯独不加ac.all[i]
      ac.all = append(ac.all[:i], ac.all[i+1:]...)
      //这里由于可能存储目标address账户的节点不止一个，需要分类讨论
      if ba := removeAccount(ac.byAddr[removed.Address], removed); len(ba) == 0 {
         delete(ac.byAddr, removed.Address)
      } else {
         ac.byAddr[removed.Address] = ba
      }
   }
}

func removeAccount(slice []accounts.Account, elem accounts.Account) []accounts.Account {
   for i := range slice {
      if slice[i] == elem {
         return append(slice[:i], slice[i+1:]...)
      }
   }
   return slice
}

// find returns the cached account for address if there is a unique match.
// The exact matching rules are explained by the documentation of accounts.Account.
// Callers must hold ac.mu.
// 如果存在唯一匹配项，则查找返回地址的缓存帐户。 accounts.Account 的文档解释了精确匹配规则。
// find是找账户的操作，首先找account,然后如果给了url那就是精确查找，那么就根据url进行匹配，这里要注意url匹配可能是非绝对地址的匹配
// 如果没有给url,如果只有一个对应的url，那么算找到了，如果0个，就是address给错了，如果有多个，那就是AmbiguousAddrError错误
func (ac *accountCache) find(a accounts.Account) (accounts.Account, error) {
   // Limit search to address candidates if possible.
   matches := ac.all
   if (a.Address != common.Address{}) {
      matches = ac.byAddr[a.Address]
   }
   if a.URL.Path != "" {
      // If only the basename is specified, complete the path.
      // 如果只指定了basename，修复路径。
      // 判断是否包含了操作系统中的操作符如’/‘,’\‘,如果不包含就是不是一个绝对路径
      // 就把他变成一个绝对路径，即ac.keydir+a.URL.Path
      if !strings.ContainsRune(a.URL.Path, filepath.Separator) {
         a.URL.Path = filepath.Join(ac.keydir, a.URL.Path)
      }
      for i := range matches {
         if matches[i].URL == a.URL {
            return matches[i], nil
         }
      }
      if (a.Address == common.Address{}) {
         return accounts.Account{}, ErrNoMatch
      }
   }
   switch len(matches) {
   case 1:
      return matches[0], nil
   case 0:
      return accounts.Account{}, ErrNoMatch
   default:
      //存在多个matches
      err := &AmbiguousAddrError{Addr: a.Address, Matches: make([]accounts.Account, len(matches))}
      copy(err.Matches, matches)
      sort.Sort(accountsByURL(err.Matches))
      return accounts.Account{}, err
   }
}

// 判断是否需要重新reload，也是需要mu.lock，然后看2秒是否更新了，体现了函数的节流，如果真的需要scanAccounts函数
func (ac *accountCache) maybeReload() {
   ac.mu.Lock()
   if ac.watcher.running {
      ac.mu.Unlock()
      return // A watcher is running and will keep the cache up-to-date.
   }
   //这里体现了函数节流，
   if ac.throttle == nil {
      ac.throttle = time.NewTimer(0)
   } else {
      select {
      //判断是否过期，过期了就不用调用了，<-ac.throttle.C会在设置的计时器结束后发送一个符号给到管道内
      case <-ac.throttle.C:
      default:
         ac.mu.Unlock()
         return // The cache was reloaded recently.
      }
   }
   // No watcher running, start it.
   ac.watcher.start()
   //2秒之内不需要reload
   ac.throttle.Reset(minReloadInterval)
   ac.mu.Unlock()
   ac.scanAccounts()
}

// 关闭操作
func (ac *accountCache) close() {
   ac.mu.Lock()
   ac.watcher.close()
   if ac.throttle != nil {
      ac.throttle.Stop()
   }
   if ac.notify != nil {
      close(ac.notify)
      ac.notify = nil
   }
   ac.mu.Unlock()
}

// scanAccounts checks if any changes have occurred on the filesystem, and
// updates the account cache accordingly
// scanAccounts 检查文件系统是否发生任何更改，并相应地更新帐户缓存
// 首先从filecache中找到变动，然后创立了一个函数查找每条变动的地址，对地址中的内容进行读取创建一个account，然后对这个account进行创建等操作
func (ac *accountCache) scanAccounts() error {
   // Scan the entire folder metadata for file changes
   // 检查是否有更改等操作产生
   creates, deletes, updates, err := ac.fileC.scan(ac.keydir)
   if err != nil {
      log.Debug("Failed to reload keystore contents", "err", err)
      return err
   }
   if creates.Size() == 0 && deletes.Size() == 0 && updates.Size() == 0 {
      return nil
   }
   // Create a helper method to scan the contents of the key files
   var (
      //bufio库的Reader，io的缓存区
      buf = new(bufio.Reader)
      key struct {
         Address string `json:"address"`
      }
   )
   readAccount := func(path string) *accounts.Account {
      //打开指定的path,读取里面key的信息
      fd, err := os.Open(path)
      if err != nil {
         log.Trace("Failed to open keystore file", "path", path, "err", err)
         return nil
      }
      defer fd.Close()
      //buf.Reset(fd)的作用是将buf的底层缓冲区切换为fd的内容，并重置所有相关的状态
      //这样可以使用bufio.Reader从fd中读取内容了。
      buf.Reset(fd)
      // Parse the address.
      key.Address = ""
      //NewDecoder从buf中读取json数据，解码放入key中
      err = json.NewDecoder(buf).Decode(&key)
      addr := common.HexToAddress(key.Address)
      switch {
      case err != nil:
         log.Debug("Failed to decode keystore key", "path", path, "err", err)
      case (addr == common.Address{}):
         log.Debug("Failed to decode keystore key", "path", path, "err", "missing or zero address")
      default:
         return &accounts.Account{Address: addr, URL: accounts.URL{Scheme: KeyStoreScheme, Path: path}}
      }
      return nil
   }
   // Process all the file diffs
   start := time.Now()
   //创建，更新，删除
   for _, p := range creates.List() {
      if a := readAccount(p.(string)); a != nil {
         ac.add(*a)
      }
   }
   for _, p := range deletes.List() {
      ac.deleteByFile(p.(string))
   }
   for _, p := range updates.List() {
      path := p.(string)
      ac.deleteByFile(path)
      if a := readAccount(path); a != nil {
         ac.add(*a)
      }
   }
   end := time.Now()
   //发送一个注意，应该是个订阅
   select {
   case ac.notify <- struct{}{}:
   default:
   }
   log.Trace("Handled keystore changes", "time", end.Sub(start))
   return nil
}
```