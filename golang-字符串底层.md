
# GoLang字符串底层
## 1. string 标准概念 
```
// string is the set of all strings of 8-bit bytes, conventionally but not
// necessarily representing UTF-8-encoded text. A string may be empty, but
// not nil. Values of string type are immutable.
type string string
```
> string是8比特字节的集合，通常但并不一定是UTF-8编码的文本
> - string可以为空（长度为0），但不会是nil
> - string对象不可以修改
>> `Go SDK v1.12.7` `src/builtin/builtin.go` 68~71



## 2. string 数据结构
源码包`src/runtime/string.go:stringStruct` 中的定义为

```
type stringStruct struct {
	str unsafe.Pointer
	len int
}
```
> 字符串其实是一个结构体，有两个私有成员：一个指针，一个int型的长度
> - stringStruct.str：字符串的首地址
> - stringStruct.len：字符串的长度
>> `Go SDK v1.12.7` `src/runtime/string.go` 218~221


另外，在reflect.StringHeader中对其运行时表现也有定义
```
// StringHeader is the runtime representation of a string.
// It cannot be used safely or portably and its representation may
// change in a later release.
// Moreover, the Data field is not sufficient to guarantee the data
// it references will not be garbage collected, so programs must keep
// a separate, correctly typed pointer to the underlying data.
type StringHeader struct {
	Data uintptr
	Len  int
}
```
> StringHeader是string（字符串）的运行时表现，它不能安全或可移植地使用，并且它的表示形式可能在以后的版本中发生更改。
> 此外，数据字段不足以保证它引用的数据不会被垃圾收集，因此程序必须保持一个单独的、正确键入的指向底层数据的指针。
> - StringHeader.Data：底层字节数组首地址，是一个不可修改的内存区域
> - StringHeader.Len：字符串字节的长度
>> `Go SDK v1.12.7` `src/reflect/value.go` 1877~1886

## 3. string 赋值操作
string在runtime包中就是stringStruct，对外呈现叫做string。构建过程是先跟据字符串构建stringStruct，再转换成string。
```
//go:nosplit
func gostringnocopy(str *byte) string { 				// 根据字符串地址构建string
    ss := stringStruct{str: unsafe.Pointer(str), len: findnull(str)} 	// 先构造stringStruct
    s := *(*string)(unsafe.Pointer(&ss))                             	// 再将stringStruct转换成string
    return s
}
```
>> `Go SDK v1.12.7` `src/runtime/string.go` 470~475
```
var str string
str = "hello world"
```

而在反射中，字符串的赋值操作也就是reflect.StringHeader结构体的复制过程，并不会涉及底层字节数组的复制。字符串赋值只是复制了数据地址和对应的长度，而不会导致底层数据的复制
```
    str := "hello, world"
    fmt.Println(str)
    fmt.Println(&str)
    
    data := [...] byte{'G','o'}
    hdr :=(*reflect.StringHeader)(unsafe.Pointer(&str))
    hdr.Data = uintptr(unsafe.Pointer(&data[0]))
    hdr.Len = len(data)
    fmt.Println(str)
    fmt.Println(&str)
```

## 4. string 拼接操作

## 5. string 其他操作

## 6. 几点疑问
### 6.1 为什么字符串不允许修改
- 在Go的实现中，string是没有自己的内存空间，只有一个指向内存的指针，这样做的好处是string变得非常轻量，可以很方便的进行传递而不用担心内存拷贝。string常量会在编译期分配到只读段，对应数据地址不可写入，并且相同的string常量不会重复存储

### 6.2 string和[]byte如何取舍
因为string直观，在实际应用中还是大量存在，在偏底层的实现中[]byte使用更多
- string 擅长的场景：
  - 需要字符串比较的场景
  - 不需要nil字符串的场景
- []byte擅长的场景：
  - 修改字符串的场景，尤其是修改粒度为1个字节
  - 函数返回值，需要用nil表示含义的场景
  - 需要切片操作的场景
