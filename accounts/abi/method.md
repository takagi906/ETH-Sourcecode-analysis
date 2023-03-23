# Method

Method.go主要提供了两个主要的函数pack以及unpack，对参数的打包，即从Int,String类型到[]byte类型，并将结构变成各类长度

```go
// 将所给的参数进行打包
func (method Method) pack(args ...interface{}) ([]byte, error) {
   // Make sure arguments match up and pack them
   // 参数个数是否一致，arg是提供的参数,inputs是method规定的参数
   if len(args) != len(method.Inputs) {
      return nil, fmt.Errorf("argument count mismatch: %d for %d", len(args), len(method.Inputs))
   }
   // variable input is the output appended at the end of packed
   // output. This is used for strings and bytes types input.
   var variableInput []byte

   var ret []byte
   for i, a := range args {

      input := method.Inputs[i]
      // pack the input
      packed, err := input.Type.pack(reflect.ValueOf(a))
      if err != nil {
         return nil, fmt.Errorf("`%s` %v", method.Name, err)
      }
      fmt.Println(input, a)
      // check for a slice type (string, bytes, slice)，如果是这三个，需要计算偏移offset
      if input.Type.requiresLengthPrefix() {
         // calculate the offset
         offset := len(method.Inputs)*32 + len(variableInput)

         // set the offset
         ret = append(ret, packNum(reflect.ValueOf(offset))...)
         // Append the packed output to the variable input. The variable input
         // will be appended at the end of the input.
         // 将打包输出附加到变量输入。 变量输入
         // 将附加在输入的末尾。
         variableInput = append(variableInput, packed...)
      } else {
         // append the packed value to the input
         ret = append(ret, packed...)
      }
   }
   // append the variable input at the end of the packed input
   ret = append(ret, variableInput...)
   return ret, nil
}
```

以及元组(结构体等)以及单独的输出，从[]byte到各种类型，这里都只提供了一个参数处理入口，具体操作都是要在pack.go里实现的

```go
// 关于tuple的分解，从output分解->v
func (method Method) tupleUnpack(v interface{}, output []byte) error {
   // make sure the passed value is a pointer

   valueOf := reflect.ValueOf(v)
   //v一定是个指针类型,否则报错
   if reflect.Ptr != valueOf.Kind() {
      return fmt.Errorf("abi: Unpack(non-pointer %T)", v)
   }
   //kind是大类型，type是更为精确的类型，kind是家电，type是电视机
   //https://swanspouse.github.io/2018/11/18/golang-reflect-type-kind-value/
   var (
      value = valueOf.Elem()
      typ   = value.Type()
   )
   j := 0
   for i := 0; i < len(method.Outputs); i++ {
      toUnpack := method.Outputs[i]
      if toUnpack.Type.T == ArrayTy {
         // need to move this up because they read sequentially
         // 需要将其向前移动，因为它们是按顺序读取的,猜测是整个数组大小也被记录在outPut中
         j += toUnpack.Type.Size
      }
      marshalledValue, err := toGoType((i+j)*32, toUnpack.Type, output)

      if err != nil {
         return err
      }
      reflectValue := reflect.ValueOf(marshalledValue)

      switch value.Kind() {
      //对value进行赋值
      case reflect.Struct:
         for j := 0; j < typ.NumField(); j++ {
            field := typ.Field(j)
            // TODO read tags: `abi:"fieldName"`
            //第一位得是大写字母如Int,String
            if field.Name == strings.ToUpper(method.Outputs[i].Name[:1])+method.Outputs[i].Name[1:] {
               if err := set(value.Field(j), reflectValue, method.Outputs[i]); err != nil {
                  return err
               }
            }
         }
      case reflect.Slice, reflect.Array:
         if value.Len() < i {
            return fmt.Errorf("abi: insufficient number of arguments for unpack, want %d, got %d", len(method.Outputs), value.Len())
         }
         v := value.Index(i)
         if v.Kind() != reflect.Ptr && v.Kind() != reflect.Interface {
            return fmt.Errorf("abi: cannot unmarshal %v in to %v", v.Type(), reflectValue.Type())
         }
         reflectValue := reflect.ValueOf(marshalledValue)
         if err := set(v.Elem(), reflectValue, method.Outputs[i]); err != nil {
            return err
         }
      default:
         return fmt.Errorf("abi: cannot unmarshal tuple in to %v", typ)
      }
   }
   return nil
}

// tuple的判断方式就是输出元素多少
func (method Method) isTupleReturn() bool { return len(method.Outputs) > 1 }

// 单独元素的Unpack与tuple差不多，首先从output里按照要求v的类型变换成对应类型的值，然后再将值赋给v
func (method Method) singleUnpack(v interface{}, output []byte) error {
   // make sure the passed value is a pointer
   valueOf := reflect.ValueOf(v)
   if reflect.Ptr != valueOf.Kind() {
      return fmt.Errorf("abi: Unpack(non-pointer %T)", v)
   }

   value := valueOf.Elem()

   marshalledValue, err := toGoType(0, method.Outputs[0].Type, output)
   if err != nil {
      return err
   }
   if err := set(value, reflect.ValueOf(marshalledValue), method.Outputs[0]); err != nil {
      return err
   }
   return nil
}
```

以及两个简单的函数，sig()和String()，sig函数的目的是有些定义不规范，如int256定义成了int，将这些不规范的函数变成标准函数，string()函数就是将method->变成一个string，类似于function Tagaki(a int,b int) constant returns(a int)

```go
// Sig 根据 ABI 规范返回方法字符串签名。
// 例子
//
// 函数 foo(uint32 a, int b) = "foo(uint32,int256)"
//
// 请注意，“int”替代了其规范表示“int256”
// 变换成标准类型的值
func (m Method) Sig() string {
   types := make([]string, len(m.Inputs))
   i := 0
   for _, input := range m.Inputs {
      types[i] = input.Type.String()
      i++
   }
   return fmt.Sprintf("%v(%v)", m.Name, strings.Join(types, ","))
}

// 将method->变成一个string，类似于function Tagaki(a int,b int) constant returns(a int)
func (m Method) String() string {
   inputs := make([]string, len(m.Inputs))
   for i, input := range m.Inputs {
      inputs[i] = fmt.Sprintf("%v %v", input.Name, input.Type)
   }
   outputs := make([]string, len(m.Outputs))
   for i, output := range m.Outputs {
      if len(output.Name) > 0 {
         outputs[i] = fmt.Sprintf("%v ", output.Name)
      }
      outputs[i] += output.Type.String()
   }
   constant := ""
   if m.Const {
      constant = "constant "
   }
   return fmt.Sprintf("function %v(%v) %sreturns(%v)", m.Name, strings.Join(inputs, ", "), constant, strings.Join(outputs, ", "))
}
```