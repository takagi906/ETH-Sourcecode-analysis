# manager

manager实现了将不同Backend的钱包按序集中起来的功能，并提供了钱包更新功能即删除或添加。以及提供了按URL查找以及按acount查找的接口。

```go
type Manager struct {
   backends map[reflect.Type][]Backend // Index of backends currently registered
   updaters []event.Subscription       // Wallet update subscriptions for all backends
   updates  chan WalletEvent           // Subscription sink for backend wallet changes
   wallets  []Wallet                   // Cache of all wallets from all registered backends

   feed event.Feed // Wallet feed notifying of arrivals/departures

   quit chan chan error
   lock sync.RWMutex
}

// NewManager creates a generic account manager to sign transaction via various
// supported backends.
// NewManager 创建一个通用的账户经理来通过各种方式签署交易支持的后端。
func NewManager(backends ...Backend) *Manager {
   // Retrieve the initial list of wallets from the backends and sort by URL
   var wallets []Wallet
   for _, backend := range backends {
      wallets = merge(wallets, backend.Wallets()...)
   }
   // Subscribe to wallet notifications from all backends
   updates := make(chan WalletEvent, 4*len(backends))

   subs := make([]event.Subscription, len(backends))
   // 进行订阅，消息通道为updates,sub代表的是各个backend,各个backend的钱包消息都通过updates传递过来
   for i, backend := range backends {
      subs[i] = backend.Subscribe(updates)
   }
   // Assemble the account manager and return
   am := &Manager{
      backends: make(map[reflect.Type][]Backend),
      updaters: subs,
      updates:  updates,
      wallets:  wallets,
      quit:     make(chan chan error),
   }
   for _, backend := range backends {
      kind := reflect.TypeOf(backend)
      am.backends[kind] = append(am.backends[kind], backend)
   }
   go am.update()

   return am
}

// Close terminates the account manager's internal notification processes.
// Close 终止客户经理的内部通知进程。
func (am *Manager) Close() error {
   errc := make(chan error)
   am.quit <- errc
   return <-errc
}

// update is the wallet event loop listening for notifications from the backends
// and updating the cache of wallets.
// update 是钱包事件循环，监听来自后端的通知并更新钱包缓存。
func (am *Manager) update() {
   // Close all subscriptions when the manager terminates
   // 关闭所有订阅
   defer func() {
      am.lock.Lock()
      for _, sub := range am.updaters {
         sub.Unsubscribe()
      }
      am.updaters = nil
      am.lock.Unlock()
   }()

   // Loop until termination
   for {
      select {
      case event := <-am.updates:
         // Wallet event arrived, update local cache
         // 就直接把钱包丢掉或者合并
         am.lock.Lock()
         switch event.Kind {
         case WalletArrived:
            am.wallets = merge(am.wallets, event.Wallet)
         case WalletDropped:
            am.wallets = drop(am.wallets, event.Wallet)
         }
         am.lock.Unlock()

         // Notify any listeners of the event
         am.feed.Send(event)

      case errc := <-am.quit:
         // Manager terminating, return
         errc <- nil
         return
      }
   }
}

// Backends retrieves the backend(s) with the given type from the account manager.
// 返回某一个type的钱包集合
func (am *Manager) Backends(kind reflect.Type) []Backend {
   return am.backends[kind]
}

// Wallets returns all signer accounts registered under this account manager.
func (am *Manager) Wallets() []Wallet {
   am.lock.RLock()
   defer am.lock.RUnlock()

   cpy := make([]Wallet, len(am.wallets))
   copy(cpy, am.wallets)
   return cpy
}

// Wallet retrieves the wallet associated with a particular URL.
func (am *Manager) Wallet(url string) (Wallet, error) {
   am.lock.RLock()
   defer am.lock.RUnlock()

   parsed, err := parseURL(url)
   if err != nil {
      return nil, err
   }
   for _, wallet := range am.Wallets() {
      if wallet.URL() == parsed {
         return wallet, nil
      }
   }
   return nil, ErrUnknownWallet
}

// Find attempts to locate the wallet corresponding to a specific account. Since
// accounts can be dynamically added to and removed from wallets, this method has
// a linear runtime in the number of wallets.
func (am *Manager) Find(account Account) (Wallet, error) {
   am.lock.RLock()
   defer am.lock.RUnlock()

   for _, wallet := range am.wallets {
      if wallet.Contains(account) {
         return wallet, nil
      }
   }
   return nil, ErrUnknownAccount
}

// Subscribe creates an async subscription to receive notifications when the
// manager detects the arrival or departure of a wallet from any of its backends.
func (am *Manager) Subscribe(sink chan<- WalletEvent) event.Subscription {
   return am.feed.Subscribe(sink)
}

// merge is a sorted analogue of append for wallets, where the ordering of the
// origin list is preserved by inserting new wallets at the correct position.
//
// The original slice is assumed to be already sorted by URL.
func merge(slice []Wallet, wallets ...Wallet) []Wallet {
   for _, wallet := range wallets {
      n := sort.Search(len(slice), func(i int) bool { return slice[i].URL().Cmp(wallet.URL()) >= 0 })
      if n == len(slice) {
         slice = append(slice, wallet)
         continue
      }
      slice = append(slice[:n], append([]Wallet{wallet}, slice[n:]...)...)
   }
   return slice
}

// drop is the couterpart of merge, which looks up wallets from within the sorted
// cache and removes the ones specified.
func drop(slice []Wallet, wallets ...Wallet) []Wallet {
   for _, wallet := range wallets {
      n := sort.Search(len(slice), func(i int) bool { return slice[i].URL().Cmp(wallet.URL()) >= 0 })
      if n == len(slice) {
         // Wallet not found, may happen during startup
         continue
      }
      slice = append(slice[:n], slice[n+1:]...)
   }
   return slice
}
```