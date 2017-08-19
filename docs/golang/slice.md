# golang 的分片

## 与数组的区别

- 数组是固定长度的，而分片确实可动态增长的，以定义为例：

```go
// 定义数组, 一定要指定长度
var names [5]string

// 定义分片, 不需要指定长度
var names []string
```

- 在函数调用时, 数组是值传递，而分片是引用传递

其实对于 `golang` 来讲，函数调用的时候都是值传递，拷贝一个副本， 之所以表现为值传递和引用传递，在于一个拷贝的是数据值，另一个拷贝的是数据指针，两个指针值指向的是同一个内存地址。

## 分片的实现

分片的底层数据还是使用的数组，它一共包含 3 个字段：

1. 地址指针
2. 长度
3. 容量

```go
// source 是一个分片， 大小为 4， 容量为： 5
source := make([]string, 4, 5)
fmt.Println(source, len(source), cap(source))
// 输出： [   ] 4 5

// 注意这里不会进行内存分配， 因为 source 还有剩余空间可以新加数据
source = append(source, "1")
fmt.Println(source, len(source), cap(source))
// 输出： [    1] 5 5
```

**在使用 `append` 来为分片添加数据时， 是否有内存分配很重要**

当 `append` 没有内存分配时：

```go
source := []string{"1", "2", "3", "4", "5"}

// 拷贝 source 的第二到第三个元素（不包括第三个）
// copied 的容量包括： 3， 4，5
copied := source[2:3]

fmt.Println("source-->", source)
fmt.Println("copied-->", copied)

fmt.Println("接下来修改 copied 的内容，看是否会对 source 产生影响...")

// 这里 append 内部不会创建一个新的底层数组，共有 source 的底层数组
// 因为 copied 的容量足够新加一个元素
// 所以不会影响到 source 的内容
copied = append(copied, "mike")
fmt.Println("source-->", source)
fmt.Println("copied-->", copied)
```

输出： 

```shell
source--> [1 2 3 4 5]
copied--> [3]
接下来修改 copied 的内容，看是否会对 source 产生影响...
source--> [1 2 3 mike 5]
copied--> [3 mike]
```

当 `append` 有内存分配时：

```go
source := []string{"1", "2", "3", "4", "5"}

// 拷贝 source 的第二到第三个元素（不包括第三个）
// copied 的容量包括： 3
copied := source[2:3:3]
// 此时 copied 会和 source 共享底层数组

fmt.Println("source-->", source)
fmt.Println("copied-->", copied)

fmt.Println("接下来修改 copied 的内容，看是否会对 source 产生影响...")

// 这里 append 内部会创建一个新的底层数组，不会共有 source 的底层数组
// 所以不会影响到 source 的内容
copied = append(copied, "mike")
fmt.Println("source-->", source)
fmt.Println("copied-->", copied)
```

输出： 

```go
source--> [1 2 3 4 5]
copied--> [3]
接下来修改 copied 的内容，看是否会对 source 产生影响...
source--> [1 2 3 4 5]
copied--> [3 mike]
```



