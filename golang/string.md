# Go 字符串

## 字符串

Go的字符串是一个任意字节的常量序列。Go语言中字符串的字节使用UTF-8编码表示Unicode文本，因此Go语言字符串是变宽字符序列，每一个字符都用一个或者多个字符表示，这跟其他的（C++，Java，Python 3）的字符串类型有着本质上的不同，后者为定宽字符序列。Go语言这样做不仅减少了内存和硬盘空间占用，同时也不用像其它语言那样需要对使用 UTF-8 字符集的文本进行编码和解码。

其他语言的字符串中的单个字符可以被字节索引，而Go中只有在字符串只包含7位的ASCII字符时才可以被字节索引。这并不代表Go在字符串处理能力上不足，因为Go语言支持一个字符一个字符的迭代，而且标准库中存在大量的字符串操作函数，最后我们还可以将Go语言的字符串转化为Unicode码点切片（类型为 [ ]rune），切片是支持直接索引的。

> 注：每一个Unicode字符都有一个唯一的叫做“码点”的标识数字。在Go语言中，一个单一的码点在内存中以 rune 的形式表示，rune表示int32类型的别名

## 定义字符串

字符串字面量使用双引号 "" 或者反引号 ` 来创建。

双引号用来创建可解析的字符串，支持转义，但不能用来引用多行；

反引号用来创建原生的字符串字面量，可能由多行组成，但不支持转义，并且可以包含除了反引号外其他所有字符。

```go
s1 := "Bourbon \nBlog \n"
s2 := `Bourbon\n
	blog\n
	`
fmt.Print(s1)
fmt.Print(s2)
```

上面代码输出为：

```go
bourbon 
Blog 
Bourbon\n
	blog\n
	
```

可见，反引号定义的字符串不但保留了换行，还保留了缩进。

双引号创建可解析的字符串应用最广泛，反引号用来创建原生的字符串则多用于书写多行消息，HTML以及正则表达式。

## 拼接字符串

虽然Go中的字符串是不可变的，但是字符串支持 + 操作和+=操作

```go
s1 := "Bourbon"
s2 := "blog"
s1 += " " + s2
fmt.Println(s1)  // Bourbon blog
```

但这种方式在处理大量字符串连接的场景下将非常低效。使用 `bytes.Buffer` 连接字符串是一个更高效的方式，它会一次性将所有的内容连接起来转化成字符串。

```go
var b bytes.Buffer
for i := 0; i < 10000; i++ {
    b.WriteString("ab")
}
s1 := b.String()
```

下面比较两种拼接操作的性能差距

```go
t := time.Now()
var b bytes.Buffer

for i := 0; i < 10000; i++ {
    b.WriteString("ab")
}
s1 := b.String()
fmt.Println(time.Now().Sub(t).String())  //345.639µs
fmt.Println(s1)
t2 := time.Now()
s2 := ""
for i := 0; i < 10000; i++ {
    s2 += "ab"
}
fmt.Println(time.Now().Sub(t2).String()) //38.053207ms
```

可见，10000次的字符串拼接会导致数量级上的性能差距。

## 类型转换

在大多数语言中，可轻易地将任意数据类型转型为字符串。但在go中强制将整形转为字符串，你不会得到期望的结果。

```go
i := 123
fmt.Println(string(i))       // {
fmt.Println(strconv.Itoa(i)) // 123
```

string()会返回整型所对应的ASCII字符，想要正确将整型转换为字符串，应当使用`strconv.Itoa()` 。反之，将字符串转换为整型可以使用`strconv.Atoi()`。

另外还可以使用`fmt.Sprintf`函数将几乎所有数据类型转换为字符串，但通常应保留在这样的实例上，如正在创建的字符串包含嵌入数据，而非在期望将单个整数转换为字符串时用。

```go
i := 123
s := fmt.Sprintf("the number is %d.", i)
fmt.Println(s)      //the number is 123.
```



## 检查前缀或后缀

在处理字符串时，想要知道一个字符串是以一个特定的字符串开始还是以一个特定的字符串结束是非常常见的情况。可以使用`strings.HasPrefix`和`strings.HasSuffix`，它们将返回一个布尔值。

```go
fmt.Println(strings.HasPrefix("something", "some"))  //true
fmt.Println(strings.HasSuffix("something", "thing")) //true
```

