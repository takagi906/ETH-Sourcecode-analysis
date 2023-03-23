# abi

abi.go是abi这个包最主要的函数，它提供了3个接口，可以看作预处理，结构函数到字节数组，字节数组重新回到结构函数的过程

pack：首先确定具体需要打包的method是需要构建的还是已经存在于已有的abi中，然后按照method的参数以及所给的参数进行对比，打包成字节数组类型。最后将方法 ID以及打包好的参数返回。event不需要打包，可能是因为是EVM调用LOG时才发生event,是内部调用，已经是二进制了，所以不需要转换

```go
func (abi ABI) Pack(name string, args ...interface{}) ([]byte, error) {
   // Fetch the ABI of the requested method
   var method Method

   if name == "" {
      method = abi.Constructor
   } else {
      m, exist := abi.Methods[name]
      if !exist {
         return nil, fmt.Errorf("method '%s' not found", name)
      }
      method = m
   }
   arguments, err := method.pack(args...)
   if err != nil {
      return nil, err
   }
   // Pack up the method ID too if not a constructor and return
   if name == "" {
      return arguments, nil
   }
   return append(method.Id(), arguments...), nil
}
```

unpack：首先判断方法是方法还是事件。方法和事件的区别：方法是表示一组操作或者功能，事件是程序运行过程中发生的某个特定的动作或状态变化。

然后对输入进来的output进行unpack，即字节数组变为一些常见的类型如int,uint.

```go
func (abi ABI) Unpack(v interface{}, name string, output []byte) (err error) {
   if err = bytesAreProper(output); err != nil {
      return err
   }
   // since there can't be naming collisions with contracts and events,
   // we need to decide whether we're calling a method or an event
   // 因为不能与合约和事件发生命名冲突，
   // 我们需要决定是调用方法还是事件
   var unpack unpacker
   if method, ok := abi.Methods[name]; ok {
      unpack = method
   } else if event, ok := abi.Events[name]; ok {
      unpack = event
   } else {
      return fmt.Errorf("abi: could not locate named method or event.")
   }

   // requires a struct to unpack into for a tuple return...
   if unpack.isTupleReturn() {
      return unpack.tupleUnpack(v, output)
   }
   return unpack.singleUnpack(v, output)
}
```

UnmarshalJSON：单纯的将json编码的输入变为一个一个的方法/事件，然后创建一个abi将这些方法组合起来

```go
func (abi *ABI) UnmarshalJSON(data []byte) error {
   var fields []struct {
      Type      string
      Name      string
      Constant  bool
      Indexed   bool
      Anonymous bool
      Inputs    []Argument
      Outputs   []Argument
   }

   if err := json.Unmarshal(data, &fields); err != nil {
      return err
   }

   abi.Methods = make(map[string]Method)
   abi.Events = make(map[string]Event)
   for _, field := range fields {
      switch field.Type {
      case "constructor":
         abi.Constructor = Method{
            Inputs: field.Inputs,
         }
      // empty defaults to function according to the abi spec
      case "function", "":
         abi.Methods[field.Name] = Method{
            Name:    field.Name,
            Const:   field.Constant,
            Inputs:  field.Inputs,
            Outputs: field.Outputs,
         }
      case "event":
         abi.Events[field.Name] = Event{
            Name:      field.Name,
            Anonymous: field.Anonymous,
            Inputs:    field.Inputs,
         }
      }
   }

   return nil
}
```
