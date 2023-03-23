# ABI概述

在doc.go中有对ETH的ABI的描述

```go
// Package abi implements the Ethereum ABI (Application Binary
// Interface).
//
// The Ethereum ABI is strongly typed, known at compile time
// and static. This ABI will handle basic type casting; unsigned
// to signed and visa versa. It does not handle slice casting such
// as unsigned slice to signed slice. Bit size type casting is also
// handled. ints with a bit size of 32 will be properly cast to int256,
// etc.
```

包 abi 实现了以太坊 ABI (Application Binary interface）。
//
以太坊 ABI 是强类型的，在编译时已知和静态。 这个 ABI 将处理基本的类型转换； 无符号到有符号，反之亦然。 它不处理切片从无符号切片到有符号切片。 但是也可以处理位大小转换。 int32 的整数将正确转换为 int256.

下面是strongly typed, known at compile time and static在chatGPT上的解释

Strongly typed, known at compile time and static 是编程语言中常用的术语，用来描述编程语言的类型系统和程序行为的特性。

Strongly typed 意味着编程语言具有强类型系统，即变量和表达式必须具有明确的数据类型。这意味着编译器在编译时可以检查数据类型，以确保程序的正确性和安全性。相对于弱类型语言，强类型语言可以更容易地发现代码中的错误和缺陷。

Known at compile time 意味着程序在编译时就已经确定了所有的变量和表达式的值，包括常量和字面量。这是因为这些值是在编译时就已经确定的，而不是在运行时动态计算的。这使得编译器可以进行更多的优化，以提高程序的性能。

Static 意味着程序的结构和行为在编译时就已经确定了，而不是在运行时动态变化的。这意味着编译器可以在编译时对程序进行更多的优化，以提高程序的性能。相对于动态语言，静态语言可以更容易地发现代码中的错误和缺陷，并且在运行时可以更好地保持性能。

个人认为这个ABI主要就是处理了各种类型转换

