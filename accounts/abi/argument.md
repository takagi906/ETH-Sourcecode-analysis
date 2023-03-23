# Argument

argument.go提供了一个数据结构以及一个常规json操作

```go
// Argument holds the name of the argument and the corresponding type.
// Types are used when packing and testing arguments.
// Argument 包含参数的名称和相应的类型。 打包和测试参数时使用类型。
type Argument struct {
   Name    string
   Type    Type
   Indexed bool // indexed is only used by events
}
// 提供了json到argument的操作
func (a *Argument) UnmarshalJSON(data []byte) error {
   var extarg struct {
      Name    string
      Type    string
      Indexed bool
   }
   err := json.Unmarshal(data, &extarg)
   if err != nil {
      return fmt.Errorf("argument json err: %v", err)
   }

   a.Type, err = NewType(extarg.Type)
   if err != nil {
      return err
   }
   a.Name = extarg.Name
   a.Indexed = extarg.Indexed

   return nil
}
```