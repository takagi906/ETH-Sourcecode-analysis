# File_cache

主要提供了一个scan函数，对提供的keyDir进行扫描并返回三个集合，分别为创建、删除、更新，各个集合中的都是各个文件的绝对路径

```go
// fileCache is a cache of files seen during scan of keystore.
type fileCache struct {
   //非线程安全的集合(取自keystore文件夹)
   all     *set.SetNonTS // Set of all files from the keystore folder
   lastMod time.Time     // Last time instance when a file was modified
   //读写锁，读时可以多个人访问，写时只能一个人访问，适合于读很多，写很少的情况
   mu sync.RWMutex
}

// scan performs a new scan on the given directory, compares against the already
// cached filenames, and returns file sets: creates, deletes, updates.
// scan 对给定目录执行新扫描，与已缓存的文件名进行比较，并返回文件集：创建、删除、更新。
func (fc *fileCache) scan(keyDir string) (set.Interface, set.Interface, set.Interface, error) {
   t0 := time.Now()

   // List all the failes from the keystore folder
   // 列出密钥库文件夹中的所有文件
   files, err := ioutil.ReadDir(keyDir)
   if err != nil {
      return nil, nil, nil, err
   }
   t1 := time.Now()

   fc.mu.Lock()
   defer fc.mu.Unlock()

   // Iterate all the files and gather their metadata
   // 迭代所有文件并收集它们的元数据

   // 所有文件绝对路径
   all := set.NewNonTS()
   // 更新过的文件的绝对路径
   mods := set.NewNonTS()
   //记录最后更新的时间
   var newLastMod time.Time
   for _, fi := range files {
      // Skip any non-key files from the folder
      path := filepath.Join(keyDir, fi.Name())
      // 判断是否应为跳过的文件
      if skipKeyFile(fi) {
         log.Trace("Ignoring file on account scan", "path", path)
         continue
      }
      // Gather the set of all and fresly modified files
      all.Add(path)

      modified := fi.ModTime()
      //modified的时间晚于fc.lastMod，则加入mods中
      if modified.After(fc.lastMod) {
         mods.Add(path)
      }
      //更新 最后更新的时间
      if modified.After(newLastMod) {
         newLastMod = modified
      }
   }
   t2 := time.Now()

   // Update the tracked files and return the three sets
   // set.difference(fc.all, all)是在fc.all中存在，但在all中不存在的元素集合
   deletes := set.Difference(fc.all, all)   // Deletes = previous - current
   creates := set.Difference(all, fc.all)   // Creates = current - previous
   updates := set.Difference(mods, creates) // Updates = modified - creates

   // 进行更新
   fc.all, fc.lastMod = all, newLastMod
   t3 := time.Now()

   // Report on the scanning stats and return
   log.Debug("FS scan times", "list", t1.Sub(t0), "set", t2.Sub(t1), "diff", t3.Sub(t2))
   return creates, deletes, updates, nil
}

// skipKeyFile ignores editor backups, hidden files and folders/symlinks.
// skipKeyFile 忽略编辑器备份、隐藏文件和文件夹/符号链接。
func skipKeyFile(fi os.FileInfo) bool {
   // Skip editor backups and UNIX-style hidden files.
   // 跳过编辑器备份和 UNIX 风格的隐藏文件。
   // HasSuffix后缀,HasPrefix前缀.
   if strings.HasSuffix(fi.Name(), "~") || strings.HasPrefix(fi.Name(), ".") {
      return true
   }
   // Skip misc special files, directories (yes, symlinks too).
   // 跳过其他特殊文件、目录（是的，还有符号链接）
   // fi.IsDir()判断是否是目录
   // fi.Mode()&os.ModeType != 0由fi.mode(fi的模式位)来知晓fi的特点(普通文件，特殊文件)，再与os.ModeType相与得到结果
   if fi.IsDir() || fi.Mode()&os.ModeType != 0 {
      return true
   }
   return false
}
```