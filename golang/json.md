# golang 读写 json 文件

## 将 struct 写入 json 文件

定义一个结构体，注意成员名大写，否则 json 无法读取。

```go
type Student struct {
	ID     int
	Name   string
	Scores []int
}
```

### 将 struct 转换成 json

```go
tom := Student{
    ID:     1,
    Name:   "Tom",
    Scores: []int{99, 100, 80, 77},
}
json, err := json.Marshal(tom)
if err != nil {
    log.Fatal(err)
}
```

### 将 json 写入文件

创建 `test.json` 文件，然后写入。

```go
err = ioutil.WriteFile("test.json", json, os.ModeAppend)
if err != nil {
    log.Fatal(err)
}
```

`test.json`: 

```json
{"ID":1,"Name":"Tom","Scores":[99,100,80,77]}
```



## 读取 json 文件转换 struct 

### 读取json 文件

```go
stu := Student{}
j, err := ioutil.ReadFile("test.json")
if err != nil {
    log.Fatal(err)
}
```

### 解析到结构体

```go
err = json.Unmarshal(j, &stu)
if err != nil {
    log.Fatal(err)
}
fmt.Print(stu)   // {1 Tom [99 100 80 77]}
```

