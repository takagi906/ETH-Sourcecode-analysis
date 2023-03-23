# Event

event中的函数与method函数基本一致，没有了pack，这猜测是因为event是EVM调用log发生的事件，不需要外部调用，所以就没有pack.

```go
// Id returns the canonical representation of the event's signature used by the
// abi definition to identify event names and types.
// Id 返回事件签名的规范表示，由 abi 定义使用以识别事件名称和类型
func (e Event) Id() common.Hash {
   types := make([]string, len(e.Inputs))
   i := 0
   // 这里的input就是指argument.go里的结构体
   for _, input := range e.Inputs {

      types[i] = input.Type.String()
      i++
   }
   //fmt.Sprintf有返回值，而fmt.Println没有返回值，直接输出。两者都是用于格式化输出的函数
   //strings.Join是把array里的所有值以某个分隔符的方式串成字符串
   return common.BytesToHash(crypto.Keccak256([]byte(fmt.Sprintf("%v(%v)", e.Name, strings.Join(types, ",")))))
}

// unpacks an event return tuple into a struct of corresponding go types
//
// Unpacking can be done into a struct or a slice/array.
// 将事件返回元组解包到相应的 go 类型的结构中
//
// 可以解包到结构或切片/数组中。
// Tuple（元组）是计算机编程语言中的一种数据结构，它可以用于存储一组不同类型的有序数据集合
func (e Event) tupleUnpack(v interface{}, output []byte) error {
   // make sure the passed value is a pointer
   valueOf := reflect.ValueOf(v)
   if reflect.Ptr != valueOf.Kind() {
      return fmt.Errorf("abi: Unpack(non-pointer %T)", v)
   }

   var (
      value = valueOf.Elem()
      typ   = value.Type()
   )

   if value.Kind() != reflect.Struct {
      return fmt.Errorf("abi: cannot unmarshal tuple in to %v", typ)
   }

   j := 0
   for i := 0; i < len(e.Inputs); i++ {
      input := e.Inputs[i]
      if input.Indexed {
         // can't read, continue
         continue
      } else if input.Type.T == ArrayTy {
         // need to move this up because they read sequentially
         j += input.Type.Size
      }
      marshalledValue, err := toGoType((i+j)*32, input.Type, output)
      if err != nil {
         return err
      }
      reflectValue := reflect.ValueOf(marshalledValue)

      switch value.Kind() {
      case reflect.Struct:
         for j := 0; j < typ.NumField(); j++ {
            field := typ.Field(j)
            // TODO read tags: `abi:"fieldName"`
            if field.Name == strings.ToUpper(e.Inputs[i].Name[:1])+e.Inputs[i].Name[1:] {
               if err := set(value.Field(j), reflectValue, e.Inputs[i]); err != nil {
                  return err
               }
            }
         }
      case reflect.Slice, reflect.Array:
         if value.Len() < i {
            return fmt.Errorf("abi: insufficient number of arguments for unpack, want %d, got %d", len(e.Inputs), value.Len())
         }
         v := value.Index(i)
         if v.Kind() != reflect.Ptr && v.Kind() != reflect.Interface {
            return fmt.Errorf("abi: cannot unmarshal %v in to %v", v.Type(), reflectValue.Type())
         }
         reflectValue := reflect.ValueOf(marshalledValue)
         if err := set(v.Elem(), reflectValue, e.Inputs[i]); err != nil {
            return err
         }
      default:
         return fmt.Errorf("abi: cannot unmarshal tuple in to %v", typ)
      }
   }
   return nil
}

func (e Event) isTupleReturn() bool { return len(e.Inputs) > 1 }

func (e Event) singleUnpack(v interface{}, output []byte) error {
   // make sure the passed value is a pointer
   valueOf := reflect.ValueOf(v)
   if reflect.Ptr != valueOf.Kind() {
      return fmt.Errorf("abi: Unpack(non-pointer %T)", v)
   }

   if e.Inputs[0].Indexed {
      return fmt.Errorf("abi: attempting to unpack indexed variable into element.")
   }

   value := valueOf.Elem()

   marshalledValue, err := toGoType(0, e.Inputs[0].Type, output)
   if err != nil {
      return err
   }
   if err := set(value, reflect.ValueOf(marshalledValue), e.Inputs[0]); err != nil {
      return err
   }
   return nil
}
```