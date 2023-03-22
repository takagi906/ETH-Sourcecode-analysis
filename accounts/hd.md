定义了出错会产生的默认路径

```go
// DefaultRootDerivationPath is the root path to which custom derivation endpoints
// are appended. As such, the first account will be at m/44'/60'/0'/0, the second
// at m/44'/60'/0'/1, etc.
var DefaultRootDerivationPath = DerivationPath{0x80000000 + 44, 0x80000000 + 60, 0x80000000 + 0, 0}

// DefaultBaseDerivationPath is the base path from which custom derivation endpoints
// are incremented. As such, the first account will be at m/44'/60'/0'/0, the second
// at m/44'/60'/0'/1, etc.
var DefaultBaseDerivationPath = DerivationPath{0x80000000 + 44, 0x80000000 + 60, 0x80000000 + 0, 0, 0}

// DefaultLedgerBaseDerivationPath is the base path from which custom derivation endpoints
// are incremented. As such, the first account will be at m/44'/60'/0'/0, the second
// at m/44'/60'/0'/1, etc.
var DefaultLedgerBaseDerivationPath = DerivationPath{0x80000000 + 44, 0x80000000 + 60, 0x80000000 + 0, 0}

// DerivationPath represents the computer friendly version of a hierarchical
// deterministic wallet account derivaion path.
//
// DerivationPath 表示分层确定性钱包帐户派生路径的计算机友好版本。
//
// The BIP-32 spec https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki
// defines derivation paths to be of the form:
//
// m / purpose' / coin_type' / account' / change / address_index
//
// The BIP-44 spec https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki
// defines that the `purpose` be 44' (or 0x8000002C) for crypto currencies, and
// SLIP-44 https://github.com/satoshilabs/slips/blob/master/slip-0044.md assigns
// the `coin_type` 60' (or 0x8000003C) to Ethereum.
//
// The root path for Ethereum is m/44'/60'/0'/0 according to the specification
// from https://github.com/ethereum/EIPs/issues/84, albeit it's not set in stone
// yet whether accounts should increment the last component or the children of
// that. We will go with the simpler approach of incrementing the last component.
```

检查用户提供的派生路径的正确性，并将它处理为计算机看得懂的路径

```go
func ParseDerivationPath(path string) (DerivationPath, error) {
   var result DerivationPath

   // Handle absolute or relative paths
   components := strings.Split(path, "/")
   //处理m
   switch {
   case len(components) == 0:
      return nil, errors.New("empty derivation path")

   case strings.TrimSpace(components[0]) == "":
      return nil, errors.New("ambiguous path: use 'm/' prefix for absolute paths, or no leading '/' for relative ones")

   case strings.TrimSpace(components[0]) == "m":
      components = components[1:]

   default:
      result = append(result, DefaultRootDerivationPath...)
   }
   // All remaining components are relative, append one by one
   if len(components) == 0 {
      return nil, errors.New("empty derivation path") // Empty relative paths
   }
   for _, component := range components {
      // Ignore any user added whitespace
      // 忽略用户输入的空格
      component = strings.TrimSpace(component)
      var value uint32

      // Handle hardened paths
      // hasSuffix 是否以某个字符结束，TrimSuffix，去除这个
      if strings.HasSuffix(component, "'") {
         value = 0x80000000
         component = strings.TrimSpace(strings.TrimSuffix(component, "'"))
      }
      // Handle the non hardened component，hardened就是末尾含'，没'不对
      // 变成big.int类型,值为component,0代表component的基数10进制
      bigval, ok := new(big.Int).SetString(component, 0)
      if !ok {
         return nil, fmt.Errorf("invalid component: %s", component)
      }
      max := math.MaxUint32 - value
      if bigval.Sign() < 0 || bigval.Cmp(big.NewInt(int64(max))) > 0 {
         if value == 0 {
            return nil, fmt.Errorf("component %v out of allowed range [0, %d]", bigval, max)
         }
         return nil, fmt.Errorf("component %v out of allowed hardened range [0, %d]", bigval, max)
      }
      value += uint32(bigval.Uint64())

      // Append and repeat
      result = append(result, value)
   }
   return result, nil
}
```

从计算机看得懂的路径又转换为用户路径，怀疑是为了测试专门写的

```go
func (path DerivationPath) String() string {
   result := "m"
   for _, component := range path {
      var hardened bool
      if component >= 0x80000000 {
         component -= 0x80000000
         hardened = true
      }
      result = fmt.Sprintf("%s/%d", result, component)
      if hardened {
         result += "'"
      }
   }
   return result
}
```

