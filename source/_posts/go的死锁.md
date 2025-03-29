---
title: go的死锁简介与预防
date: 2023-07-17 00:00:00
categories:
- golang
---

## 什么是死锁

死锁是一个计算机名词，顾名思义，就是当两个或者多个进程，互相都在等待对方停止执行然后自己以获得一些资源，但是没有一方提前停止时，那么就会死锁。类比到生活中的一个例子，在小路后两个人迎面撞上，但是两个人都不给对方让路呆在原地想等对方先让路，那么这个时候就产生了死锁。

go中的死锁很大概率都是在涉及到goroutine、编写了goroutine代码的时候发生的，但是即使不涉及goroutine，也可能会发生死锁。

## 示例

下面是一个简单的golang中会发生死锁的例子：

```go
func main() {
	a := make(chan string, 10)
	go func() {
		a <- "string"
		a <- "ccc"
		// close(a)
	}()
	for v := range a {
		fmt.Println(v)
	}
}
```

运行这段代码得到的结果是：

```shell
string
ccc
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
main.main()
        /Users/rhettnina/MyLocalFile/Work/code/flora-gopacket-service/a_my_tester/aaa/main.go:15 +0xce
exit status 2
```

报错结果的15行就是`for v := range a {`这一行。

> 代码里的a是大小为10的channel，大小为10意味着在被接收前该channel最多可存储10个元素。如果代码里的a是无缓冲的channel即这样定义：`a := make(chan string)`，运行后也还说会报相同的错误。

### 发生原因

那么这段代码为什么会发生死锁呢？

因为`for v := range a {`会遍历channel a，获取到channel中的元素。往a中传入了string和ccc这两个值后都可以在15行成功获取到这两个值并打印出来，但是该程序的子goroutine往a中传入这两个值之后就退出了，之后channel a不会再被传入任何值了。而15行却不知道channel a不会再被传入任何值了，会一直等待，这种情况就是死锁了。因为golang语言自带判断死锁的机制，所以运行时如果判断出发生了死锁就会报错，所以就输出了`fatal error: all goroutines are asleep - deadlock!`这一行。

### 如何解决

解决方法也很简单，只要不让主goroutine里的`for v := range a {`这一行一直在等待即可，这一行代码一直在等待是因为主goroutine不知道channel a已经不会再有元素写入了，所以只需要告知主goroutine这一点。

解决方法是关闭掉channel，即取消注释`// close(a)`这一行。channel的关闭操作只能由channel的发送数据方操作，关闭后，读取channel数据方就会知晓channel已经被关闭了，那么接收方读取完channel的最后一个数据后就不会再陷入等待而是不再继续读取了。

总结下来，close channel的作用就是告诉接收方：这个channel已经被关闭了，不可能再有数据被写入了，你不用一直等待了。所以channel关闭后不可写但可读。

## 如何检测

其实死锁这个问题大概率是写代码的时候不会意识到，只有在程序运行后报错了才知道原来发生死锁了。报错后可用通过pprof来协助排查和定位死锁问题。所以pprof是出现问题后的检测工具，那么有没有在开发的时候就能帮助排查程序是否可能出现死锁问题的工具呢？答案就是[goleak](https://github.com/uber-go/goleak)。

## 设计最佳实践

下面是一些编码时的规范，遵循这些规范可在很大层面上避免死锁的发生。

- 如果想减少goroutine相互等待的可能性，可以用无缓冲的channel
- 为了避免一些goroutine的无限制等待，可以使用context来在goroutine之间发送和传递取消信号
- 使用select语句来实现超时效果，在select中通过time.After指定超时时间
- 无缓冲的管道应该先接收再发送
- channel尽可能进行关闭操作，这个只能发送端来做
- channel接收端应该尽可能增加对channel是否关闭了的判断

