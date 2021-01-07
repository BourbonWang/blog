# container/list 包学习笔记

golang 的链表list的使用和具体实现。list对存储的元素并没有类型限制，比较方便。实现方式值得学习借鉴。

## 函数和功能

首先列出节点和链表的成员函数和功能。

### 节点

- **Next()** ：前趋指针
- **Prev()** ：后趋指针

### 链表

- **Init()** ：初始化链表
- **Len()** ：链表长度
- **Front()** ：返回第一个节点的指针
- **Back()** ：返回最后一个节点的指针 
- **Remove(e)** ：删除节点e，返回e的值
- **PushFront(value)** ：将value加入链表头部，返回该节点
- **PushBack(value)** ：将value加入链表尾部，返回该节点
- **InsertBefore(value, e)** ：将value插入节点e之前，返回该节点 
- **InsertAfter(value, e)** ：将value插入节点e之后，返回该节点 
- **MoveToFront(e)** ：将节点e移动到链表头部
- **MoveToBack(e)** ：将节点e移动到链表尾部
- **MoveBefore(e, mark)** ：将节点e移动到节点mark前面
- **MoveAfter(e, mark)** ：将节点e移动到节点mark后面
- **PushBackList(list)** ：将另一个链表list连接到本链表尾部
- **PushFrontList(list)** ：将另一个链表list连接到本链表头部

## 使用 

### 创建链表

```go
//使用提供的New()进行初始化
l := list.New()
//使用var关键字
var l list.List
```

### 操作

```go
l := list.New()
l.PushBack(1)            // 1
e := l.PushBack(2)       // 1,2
l.PushFront(3)           // 3,1,2
e = l.InsertBefore(4,e)  // 3,1,4,2
e = l.InsertAfter(5,e)   // 3,1,4,5,2
l.MoveToBack(e)          // 3,1,4,2,5

l2 := list.New()
l2.PushBack(7)
l2.PushBack(8)
l2.PushBack(9)           //l2: 7,8,9

l.PushBackList(l2)       //l: 3,1,4,2,5,7,8,9
l.PushFrontList(l2)      //l: 7,8,9,3,1,4,2,5,7,8,9
```

### 遍历链表

```go
for e := l.Front(); e != nil; e = e.Next() {
		// do something with e.Value
}
```

##  数据结构实现

golang 的 list 使用双向环形链表实现。用一个root节点同时表示头节点和尾节点，第一个元素root.next，最后一个元素root.prev。root本身不存储数据。

### 节点

首先看节点的数据结构，list存放节点属于的链表，用来检验操作的节点是否属于该链表，避免非法传参。value存储值，可以为任何类型。

```go
type Element struct {
	next, prev *Element
	// The list to which this element belongs.
	list *List
	// The value stored with this element.
	Value interface{}
}
```

成员函数有Next()和Prev():

```go
func (e *Element) Next() *Element {
	if p := e.next; e.list != nil && p != &e.list.root {
		return p
	}
	return nil
}
```

### 链表

```go
type List struct {
	root Element 
	len  int    
}
```

元素的操作依靠root节点。除了root节点外，len存储节点个数，不包括root。

初始化

```go
func (l *List) Init() *List {
	l.root.next = &l.root
	l.root.prev = &l.root
	l.len = 0
	return l
}

// New returns an initialized list.
func New() *List { return new(List).Init() }
```

Front()和Back()可以看出root节点同时作为头节点和尾节点

```go
func (l *List) Front() *Element {
	if l.len == 0 {
		return nil
	}
	return l.root.next
}

func (l *List) Back() *Element {
	if l.len == 0 {
		return nil
	}
	return l.root.prev
}
```

### 插入

对于插入操作，首先实现了节点到节点的插入，然后在其上封装了value到节点的插入，可用于之后的各种插入操作。

```go
func (l *List) insert(e, at *Element) *Element {
	e.prev = at
	e.next = at.next
	e.prev.next = e
	e.next.prev = e
	e.list = l
	l.len++
	return e
}

func (l *List) insertValue(v interface{}, at *Element) *Element {
	return l.insert(&Element{Value: v}, at)
}
```

所以可以容易得到PushFront()和PushBack()的实现

```go
func (l *List) PushFront(v interface{}) *Element {
	l.lazyInit()
	return l.insertValue(v, &l.root)
}

func (l *List) PushBack(v interface{}) *Element {
	l.lazyInit()
	return l.insertValue(v, l.root.prev)
}
```

注意这里使用了懒加载，在第一次使用到的时候再进行初始化

```go
func (l *List) lazyInit() {
	if l.root.next == nil {
		l.Init()
	}
}
```

InsertBefore()和InsertAfter()同理，要记得检验节点是否属于该链表

```go
func (l *List) InsertBefore(v interface{}, mark *Element) *Element {
	if mark.list != l {
		return nil
	}
	return l.insertValue(v, mark.prev)
}
```

PushBackList()和PushFrontList()是合并链表的操作。同样依赖于基本的插入操作。对传入的链表进行遍历，插入到对应的位置。

```go
func (l *List) PushBackList(other *List) {
	l.lazyInit()
	for i, e := other.Len(), other.Front(); i > 0; i, e = i-1, e.Next() {
		l.insertValue(e.Value, l.root.prev)
	}
}

func (l *List) PushFrontList(other *List) {
	l.lazyInit()
	for i, e := other.Len(), other.Back(); i > 0; i, e = i-1, e.Prev() {
		l.insertValue(e.Value, &l.root)
	}
}
```

### 移除

移除操作的基本操作remove()，双链表的移除，把没用的指针设为nil

```go
func (l *List) remove(e *Element) *Element {
	e.prev.next = e.next
	e.next.prev = e.prev
	e.next = nil // avoid memory leaks
	e.prev = nil // avoid memory leaks
	e.list = nil
	l.len--
	return e
}
```

然后封装得到Remove()，同样应当检查要删除的节点是否属于该链表

```go
func (l *List) Remove(e *Element) interface{} {
	if e.list == l {
		l.remove(e)
	}
	return e.Value
}
```

### 移动

实现节点到节点的移动

```go
func (l *List) move(e, at *Element) *Element {
	if e == at {
		return e
	}
	e.prev.next = e.next
	e.next.prev = e.prev

	e.prev = at
	e.next = at.next
	e.prev.next = e
	e.next.prev = e

	return e
}
```

然后基于这个可以实现各种移动操作

```go
func (l *List) MoveToFront(e *Element) {
	if e.list != l || l.root.next == e {
		return
	}
	l.move(e, &l.root)
}

func (l *List) MoveToBack(e *Element) {
	if e.list != l || l.root.prev == e {
		return
	}
	l.move(e, l.root.prev)
}

func (l *List) MoveBefore(e, mark *Element) {
	if e.list != l || e == mark || mark.list != l {
		return
	}
	l.move(e, mark.prev)
}

func (l *List) MoveAfter(e, mark *Element) {
	if e.list != l || e == mark || mark.list != l {
		return
	}
	l.move(e, mark)
}
```

