# 详解 Go slice

切片 slice 是golang的复合类型，是对数组的补充。数组的长度不可变，go 提供了 slice 作为“动态数组”使用。 slice 底层依赖于数组。且支持通过 append() 向slice中追加元素，长度不够时会动态扩展，通过再次slice切片，可以得到得到更小的slice结构，可以迭代、遍历等。

## slice 的存储结构

slice的底层结构由3部分组成：

- pointer：指向底层数组某个元素的指针
- Length：slice当前的长度。追加元素时，长度会扩展，最大扩展到Capacity
- Capacity：底层数组的长度。由于slice的存储依赖于底层数组，所以Capacity也表示了slice最大能扩展到的长度

以上每部分占用8字节，所以一个slice都是24字节。

因此可以知道 golang 创建slice的过程：先创建一个有特定长度和数据类型的底层数组，然后从这个底层数组中选取一部分元素，返回这些元素组成的集合，并将 slice 指向集合中的第一个元素。换句话说，**slice自身维护了一个指针属性，指向它底层数组中的某些元素的集合**。

```go
a := make([]int,3,5)
fmt.Println(a)         //[0 0 0]
fmt.Println(len(a))    // 3
fmt.Println(cap(a))    // 5
println(a)             //[3/5]0xc000094030
```

上面建立了一个长度为3的切片，它的底层数组长度为5。通过println()可以看出slice的结构，`[3/5]`表示length 和 capacity， `0xc000094030` 表示指向底层数组的指针。这个slice的存储示意图：

```
|---------|----------|--------|
| pointer | capacity | length |  slice：指向底层数组的第0个元素，长度为3 
|         |    5     |    3   |
|-|-------|----------|--------|
 \|/
|-|-----------------|             array:长度为5,初始值为0的数组 
| 0 | 0 | 0 | 0 | 0 |
|---|---|---|---|---|
```

## 创建，初始化

直接创建

```go
a := []int{1, 2, 3} //创建len和cap都为3的slice，并赋值
b := []int{9: 3}    //创建len和cap都为10的slice，并将index=9的元素赋值3
                    // [0 0 0 0 0 0 0 0 0 3]
```

使用make()。可以先为底层数组分配好内存，然后从这个底层数组中再额外生成一个slice并初始化。

```go
slice := make([]int,5)   // 创建一个len和cap都为5的slice
slice := make([]int,3,5) // 创建一个len=3,cap=5的slice
```

## 空 slice 和 nil slice

当声明一个slice，但不做初始化的时候，这个slice就是一个nil slice。

```go
var nil_slice []int
```

nil slice表示它的指向底层数组的指针为nil。也因此，nil slice的长度和容量都为0。

empty slice表示长度为0，容量为0，但却有指向的底层数组，只不过暂时是长度为0的空数组。

```go
empty_slice := make([]int,0)
empty_slice := []int{}
```

无论是nil slice还是empty slice，都可以对它们进行操作，如append()、len()和cap()。

## 对数组进行切片

对数组切片即为将slice的指针指向该数组，并利用length截取数组某一部分。

```go
numbers := [9]int{0, 1, 2, 3, 4, 5, 6, 7, 8}  //len=9 cap=9 
slice1 := numbers[1:4]                        //len=3 cap=8 [1 2 3] 
slice2 := numbers[4:]                     //len=5 cap=5 slice=[4 5 6 7 8]
```

可见，底层数组始终为numbers，cap为剩余可用的数组容量。由于slice的指针指向的数组下标不同，导致了各自可用的cap不同。

因此，对于底层数组容量为k的切片`slice[i : j]`来说，len = j - i，cap = k - i。

改变切片元素，底层数组同时改变。改变数组元素，与其关联的切片随之改变。

```go
numbers[2] = 10
fmt.Println(slice1)   // [1 10 3]
slice2[3] = 10
fmt.Println(numbers)  // [0 1 10 3 4 5 6 10 8]
```

当多个slice共享同一个底层数组时，如果修改了某个slice中的元素，实际上修改的是底层数组的值，其它与之关联的slice也会随之改变。当同一个底层数组有很多slice的时候，一切将变得混乱不堪，因为我们不可能记住谁在共享它。所以，需要一种特性，保证各个slice的底层数组互不影响，相关内容见下面的"扩容"。

## append()

追加元素到slice末尾。len会增加。当追加元素后的slice仍然未达到cap时，append()新元素将赋值给底层数组。当达到cap时，扩容机制将创建新的底层数组。

append()也可以用来合并slice。append()最多允许两个参数，所以一次性只能合并两个slice。但可以将append()作为另一个append()的参数，从而实现多级合并。

```go
s1 := []int{1, 2}
s2 := []int{3, 4}
s3 := append(s1, s2...)                //len=4 cap=4 [1 2 3 4]
s4 := append(append(s1, s3...), s2...) //len=8 cap=12 
                                       //slice=[1 2 1 2 3 4 3 4]
```



## 扩容

当新的len 等于cap时，继续追加元素将引发扩容机制，开辟新的底层数组，将原来的数据复制过去。旧的底层数组仍然会被旧slice引用，新slice和旧slice不再共享同一个底层数组。

扩容的容量将进行如下判断：

1. 首先判断，如果新申请容量（cap）大于2倍的旧容量（old.cap），最终容量（newcap）就是新申请的容量（cap）。
2. 否则判断，如果旧切片的长度小于1024，则最终容量(newcap)就是旧容量(old.cap)的两倍。
3. 否则判断，如果旧切片长度大于等于1024，则最终容量（newcap）从旧容量（old.cap）开始循环增加原来的1/4，即旧容量的1.25倍、1.5倍、1.75倍……直到最终容量（newcap）大于等于新申请的容量(cap)。
4. 如果最终容量计算值溢出，则最终容量就是新申请容量(cap)。

```go
my_slice := []int{1, 2, 3, 4, 5}
// 限定长度和容量，且让长度和容量相等
new_slice := my_slice[1:3:3]       // len=2 cap=2 [2 3] 
// 扩容
app_slice := append(new_slice, 44) // len=3 cap=4 [2 3 44]
```

当限定了slice的长度和容量时，如果需要扩容，golang将生成新的底层数组，而不是对原来的数组进行扩展，因此此时的新元素不会改变原来底层数组的值。

这样就可以解决上面提到的问题，因为创建了新的底层数组，所以修改不同的slice，将不会互相影响。为了保证每次都是修改各自的底层数组，通常会切出仅一个长度、仅一个容量的新slice，这样只要对它进行任何一次扩容，就会生成一个新的底层数组，从而让每个slice的底层数组都独立。

## copy ()

copy(dst, src) 可以将src slice拷贝到dst slice。src比dst长，就截断; src比dst短，则只拷贝src那部分。这里的长度指len而不是容量cap。

copy的返回值是拷贝成功的元素数量，所以也就是src slice或dst slice中最小的那个长度。

```go
s1 := []int{3, 4, 5}
s2 := make([]int, 2, 7)
copy(s1, s2)            //s2: len=2 cap=7 [3 4]
```

## slice做函数参数

前面说过，slice的数据结构类似于`[3/5]0xc000094030`，仍可以将slice看作一种指针。这个特性直接体现在函数参数传值上。

Go中函数的参数是按值传递的，所以调用函数时会复制一个参数的副本传递给函数。如果传递给函数的是slice，它将复制该slice副本给函数，这个副本仍然是`[3/5]0xc42003df10`，它仍然指向源slice的底层数组。

如果函数内部对slice进行了修改，有可能会直接影响函数外部的底层数组，从而影响其它slice。但并不总是如此，例如函数内部对slice进行扩容，扩容时生成了一个新的底层数组，函数后续的代码只对新的底层数组操作，这样就不会影响原始的底层数组。

```go
package main

import "fmt"

func main() {
	s1 := make([]int, 3, 4)  // [0 0 0]
	foo(s1)
	printSlice(s1)           // len=3 cap=4 [10 10 10]
}

func foo(s []int) {
	for i, _ := range s {   //修改slice的值，将改变底层数组
		s[i] += 10
	}
	s = append(s, 3)        //扩容将创建新的底层数组
	s = append(s, 4)        // len=5 cap=8 [10 10 10 3 4]
	s[1] = 20               //此时不会改变底层数组
	printSlice(s)           // len=5 cap=8 [10 20 10 3 4]
}

func printSlice(x []int) {
	fmt.Printf("len=%d cap=%d slice=%v\n", len(x), cap(x), x)
}
```

