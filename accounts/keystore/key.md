# key

key.go主要实现了key编码成json和解码，如何生成一把钥匙，过程是随机生成私钥组装成钥匙。

存储过程先留个坑，因为主要的函数都是在key_store里

```go
const (
   version = 3
)

type Key struct {
   //类型为uuid.UUID，用于存储版本4的随机UUID，用于唯一标识此Key
   Id uuid.UUID // Version 4 "random" for unique id not derived from key data
   // to simplify lookups we also store the address
   // 为了方便查找，存储用于存储此Key所对应的以太坊地址
   Address common.Address
   // we only store privkey as pubkey/address can be derived from it
   // privkey in this struct is always in plaintext
   //我们只存储私钥，因为可以从中导出公钥/地址
   //此结构中的私钥始终为明文
   PrivateKey *ecdsa.PrivateKey
}

type keyStore interface {
   // Loads and decrypts the key from disk.
   // 从磁盘加载并解密密钥
   GetKey(addr common.Address, filename string, auth string) (*Key, error)
   // Writes and encrypts the key.
   // 写入并加密密钥
   StoreKey(filename string, k *Key, auth string) error
   // Joins filename with the key directory unless it is already absolute.
   // 将文件名与密钥目录连接起来（如果文件名不是绝对路径）
   JoinPath(filename string) string
}

// 不同的JSON格式的结构体类型，用于在keyStore中序列化/反序列化密钥
type plainKeyJSON struct {
   Address    string `json:"address"`
   PrivateKey string `json:"privatekey"`
   Id         string `json:"id"`
   Version    int    `json:"version"`
}

type encryptedKeyJSONV3 struct {
   Address string     `json:"address"`
   Crypto  cryptoJSON `json:"crypto"`
   Id      string     `json:"id"`
   Version int        `json:"version"`
}

type encryptedKeyJSONV1 struct {
   Address string     `json:"address"`
   Crypto  cryptoJSON `json:"crypto"`
   Id      string     `json:"id"`
   Version string     `json:"version"`
}

type cryptoJSON struct {
   Cipher       string                 `json:"cipher"`
   CipherText   string                 `json:"ciphertext"`
   CipherParams cipherparamsJSON       `json:"cipherparams"`
   KDF          string                 `json:"kdf"`
   KDFParams    map[string]interface{} `json:"kdfparams"`
   MAC          string                 `json:"mac"`
}

type cipherparamsJSON struct {
   IV string `json:"iv"`
}

// 对Key编码传输,json编码方式为先构造对应对应的结构体，再将结构体放入->json.marshal中
// 这里注意 JSON 只支持 string，因此k.address，k.PrivateKey，k.Id需要转换成string才行
func (k *Key) MarshalJSON() (j []byte, err error) {
   jStruct := plainKeyJSON{
      hex.EncodeToString(k.Address[:]),
      hex.EncodeToString(crypto.FromECDSA(k.PrivateKey)),
      k.Id.String(),
      version,
   }
   j, err = json.Marshal(jStruct)
   return j, err
}

// 对j进行解码，存入k中。json解码方式为首先json.unmarshal()->keyJSON，然后对各个元素进行转换变成对应的格式
func (k *Key) UnmarshalJSON(j []byte) (err error) {
   keyJSON := new(plainKeyJSON)
   err = json.Unmarshal(j, &keyJSON)
   if err != nil {
      return err
   }

   u := new(uuid.UUID)
   *u = uuid.Parse(keyJSON.Id)
   k.Id = *u
   addr, err := hex.DecodeString(keyJSON.Address)
   if err != nil {
      return err
   }
   privkey, err := crypto.HexToECDSA(keyJSON.PrivateKey)
   if err != nil {
      return err
   }

   k.Address = common.BytesToAddress(addr)
   k.PrivateKey = privkey

   return nil
}

// 私钥生成key
func newKeyFromECDSA(privateKeyECDSA *ecdsa.PrivateKey) *Key {
   //ID随机生产的，address私钥生成的
   id := uuid.NewRandom()
   key := &Key{
      Id:         id,
      Address:    crypto.PubkeyToAddress(privateKeyECDSA.PublicKey),
      PrivateKey: privateKeyECDSA,
   }
   return key
}

// NewKeyForDirectICAP generates a key whose address fits into < 155 bits so it can fit
// into the Direct ICAP spec. for simplicity and easier compatibility with other libs, we
// retry until the first byte is 0.
// NewKeyForDirectICAP 生成一个密钥，其地址小于 155 位，因此它可以符合 Direct ICAP 规范。 为了简单和与其他库更容易兼容，我们重试直到第一个字节为 0。
// NewKeyForDirectICAP需要满足以太坊网络发送的地址格式，以 "XE" 或 "0x00" 开头的地址
func NewKeyForDirectICAP(rand io.Reader) *Key {
   //这里不太理解为什么要新建一个randBytes，从randBytes里读数据而不是直接从rand里读
   randBytes := make([]byte, 64)
   _, err := rand.Read(randBytes)
   if err != nil {
      panic("key generation: could not read from random source: " + err.Error())
   }
   reader := bytes.NewReader(randBytes)
   privateKeyECDSA, err := ecdsa.GenerateKey(crypto.S256(), reader)
   if err != nil {
      panic("key generation: ecdsa.GenerateKey failed: " + err.Error())
   }
   key := newKeyFromECDSA(privateKeyECDSA)
   if !strings.HasPrefix(key.Address.Hex(), "0x00") {
      return NewKeyForDirectICAP(rand)
   }
   return key
}

// 直接生成一个newkey
func newKey(rand io.Reader) (*Key, error) {
   privateKeyECDSA, err := ecdsa.GenerateKey(crypto.S256(), rand)
   if err != nil {
      return nil, err
   }
   return newKeyFromECDSA(privateKeyECDSA), nil
}

// 这主要是在keystore里实现的
func storeNewKey(ks keyStore, rand io.Reader, auth string) (*Key, accounts.Account, error) {
   key, err := newKey(rand)
   if err != nil {
      return nil, accounts.Account{}, err
   }
   a := accounts.Account{Address: key.Address, URL: accounts.URL{Scheme: KeyStoreScheme, Path: ks.JoinPath(keyFileName(key.Address))}}
   if err := ks.StoreKey(a.URL.Path, key, auth); err != nil {
      zeroKey(key.PrivateKey)
      return nil, a, err
   }
   return key, a, err
}

// 往指定的file路径下写入content
// 过程首先创建文件夹，然后创建一个临时文件，如果临时文件写入失败则删除文件，否则将临时文件改名
func writeKeyFile(file string, content []byte) error {
   // Create the keystore directory with appropriate permissions
   // in case it is not present yet.
   // 创建具有适当权限的密钥库目录
   // 如果它还不存在。
   const dirPerm = 0700
   if err := os.MkdirAll(filepath.Dir(file), dirPerm); err != nil {
      return err
   }
   // Atomic write: create a temporary hidden file first
   // then move it into place. TempFile assigns mode 0600.
   f, err := ioutil.TempFile(filepath.Dir(file), "."+filepath.Base(file)+".tmp")
   if err != nil {
      return err
   }
   if _, err := f.Write(content); err != nil {
      f.Close()
      os.Remove(f.Name())
      return err
   }
   f.Close()
   return os.Rename(f.Name(), file)
}

// keyFileName implements the naming convention for keyfiles:
// UTC--<created_at UTC ISO8601>-<address hex>
// keyFileName 实现密钥文件的命名约定：UTC--<created_at UTC ISO8601>-<address hex>
func keyFileName(keyAddr common.Address) string {
   ts := time.Now().UTC()
   return fmt.Sprintf("UTC--%s--%s", toISO8601(ts), hex.EncodeToString(keyAddr[:]))
}

// 标准时间计算方法
func toISO8601(t time.Time) string {
   var tz string
   name, offset := t.Zone()
   if name == "UTC" {
      tz = "Z"
   } else {
      tz = fmt.Sprintf("%03d00", offset/3600)
   }
   return fmt.Sprintf("%04d-%02d-%02dT%02d-%02d-%02d.%09d%s", t.Year(), t.Month(), t.Day(), t.Hour(), t.Minute(), t.Second(), t.Nanosecond(), tz)
}
```