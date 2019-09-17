
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

## 4. string 常用操作
主要是strings和strconv两个包
- strings
  - strings.Index, strings.LastIndex 首次出现的位置和末次出现的位置
  - strings.ToLower, strings.ToUpper 转换大小写
  - strings.HasPrefix, strings.Suffix 前缀后缀
  - strings.Trim,strings.TrimSpace,strings.TrimLeft,strings.TrimRight 去除首尾字符
  - strings.Field,strings.Split 按空格或指定子串分割成切片
  - strings.Count,strings.Repeat,strings.Replace 子串计数，重复和替换
  - strings.Join 拼接字符串
    - `+` 直接连接字符串 
    - fmt.Sprintf 格式化字符串
    - bytes.Buffer  大量数据拼接推荐使用
- strconv  数据类型转换
  - strconv.Itoa,strconv.Atoi 整数和字符串的相互转换
  - strconv.ParseBool("true"),strconv.ParseFloat("3.1415", 64),strconv.ParseInt("-42", 10, 64) Parse类方法：转换字符串为给定类型的值
  - strconv.FormatBool()、strconv.FormatFloat()、strconv.FormatInt()、strconv.FormatUint() Format类方法：将给定类型格式化为string类型
  - strconv.AppendBool()、strconv.AppendFloat()、strconv.AppendInt()、strconv.AppendUint() Append类方法：将转换后的结果追加到一个slice中

## 5. 几点疑问
### 5.1 string和[]byte的如何转换
- []byte转换到string 会首先按切片的长度申请内存，然后构建string，`string.str=内存地址; string.len=切片长度;`,最后拷贝切片数据到新内存空间
- string转换[]byte,需要申请切片内存空间，然后把string的数据逐个拷贝到切片的新空间
### 5.2 为什么字符串不允许修改
- 在Go的实现中，string是没有自己的内存空间，只有一个指向内存的指针，这样做的好处是string变得非常轻量，可以很方便的进行传递而不用担心内存拷贝。string常量会在编译期分配到只读段，对应数据地址不可写入，并且相同的string常量不会重复存储
### 5.3 string和[]byte如何取舍
因为string直观，在实际应用中还是大量存在，在偏底层的实现中[]byte使用更多
- string 擅长的场景：
  - 需要字符串比较的场景
  - 不需要nil字符串的场景
- []byte擅长的场景：
  - 修改字符串的场景，尤其是修改粒度为1个字节
  - 函数返回值，需要用nil表示含义的场景
  - 需要切片操作的场景
### 5.4 []byte和[]rune的区别
```
// byte is an alias for uint8 and is equivalent to uint8 in all ways. It is
// used, by convention, to distinguish byte values from 8-bit unsigned
// integer values.
type byte = uint8

// rune is an alias for int32 and is equivalent to int32 in all ways. It is
// used, by convention, to distinguish character values from integer values.
type rune = int32
```
- byte，等同于uint8，可以理解为是用来存储ASCII码的类型别名
- rune，等同于int32，是为了给Unicode或utf-8编码的字符存储的类型别名，处理含有中文字符串时，推荐使用
```
str1 := "hello 世界"
fmt.Println("rune:", len([]rune(str1)))   // rune:8
fmt.Println("byte:", len([]byte(str1)))   // byte:12
```
