---
title: 实现String方法来自定义打印结果
date: 2021-3-22 00:00:00
tags:
- 原创
categories:
- golang
---

某个自定义类型，如果直接打印的话，就是其原始值，比如：

```go
type T int

func main() {
	t1 := T(1)
	fmt.Println(t1)
	t2 := T(2)
	fmt.Println(&t2)
}
```

打印结果为

```shell
1
0xc00012a010
```

 如果想自定义打印结果，比如t1打印出来是one，那么**需要这个自定义类型实现 String() string**，比如：

```go
type T int

func main() {
	t1 := T(1)
	fmt.Println(t1)
	t2 := T(2)
	fmt.Println(&t2)
}

func (t T) String() string {
	if t == T(1) {
		return "one"
	}
	return "unknown"
}
```

打印结果为

```go
one
unknown
```

另一个例子：

```go
type T struct {
	Value int
}

func main() {
	t1 := T{1}
	fmt.Println(t1)
	t2 := &T{2}
	fmt.Println(t2)
	fmt.Println(&t2)
}

func (t T) String() string {
	if t.Value == 1 {
		return "one"
	}
	return "unknown"
}
```

打印结果为

```shell
one
unknown
0xc00000e030
```

对于指针类型的t2，注意这里打印t2和&t2的不同，打印t2的时候也调用String()。


