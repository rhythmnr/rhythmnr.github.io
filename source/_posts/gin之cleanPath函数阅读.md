---
title: gin之cleanPath函数阅读
date: 2024-1-17 00:00:00
tags:
- 原创
categories:
- golang
---

最近在阅读gin框架的源代码，发现gin里有一个挺重要的函数cleanPath，这个函数和path.Clean类似，但是是url的实现方式，和go的path.Clean源码的实现方式还是有些不一样的。但是主要的功能就是给定一个路径，返回这个路径计算之后的结果，主要就是对一些相对路径如`.`，`..`的处理，比如输入`/a/b/../d`，那么就输出计算后的路径：`/a/d`，输入`/a/b/c/.`，输出`/a/b/c`。

下面是gin里cleanPath的实现，这个函数在`path.go`文件里：

```go
// cleanPath is the URL version of path.Clean, it returns a canonical URL path
// for p, eliminating . and .. elements.
//
// The following rules are applied iteratively until no further processing can
// be done:
//  1. Replace multiple slashes with a single slash.
//  2. Eliminate each . path name element (the current directory).
//  3. Eliminate each inner .. path name element (the parent directory)
//     along with the non-.. element that precedes it.
//  4. Eliminate .. elements that begin a rooted path:
//     that is, replace "/.." by "/" at the beginning of a path.
//
// If the result of this process is an empty string, "/" is returned.
func cleanPath(p string) string {
	const stackBufSize = 128
	// Turn empty string into "/"
	if p == "" {
		return "/"
	}

	// Reasonably sized buffer on stack to avoid allocations in the common case.
	// If a larger buffer is required, it gets allocated dynamically.
	buf := make([]byte, 0, stackBufSize)

	n := len(p)

	// Invariants:
	//      reading from path; r is index of next byte to process.
	//      writing to buf; w is index of next byte to write.

	// path must start with '/'
	r := 1
	w := 1

	if p[0] != '/' {
		r = 0

		if n+1 > stackBufSize {
			buf = make([]byte, n+1)
		} else {
			buf = buf[:n+1]
		}
		buf[0] = '/'
	}

	trailing := n > 1 && p[n-1] == '/'

	// A bit more clunky without a 'lazybuf' like the path package, but the loop
	// gets completely inlined (bufApp calls).
	// loop has no expensive function calls (except 1x make)		// So in contrast to the path package this loop has no expensive function
	// calls (except make, if needed).

	for r < n {
		switch {
		case p[r] == '/':
			// empty path element, trailing slash is added after the end
			r++

		case p[r] == '.' && r+1 == n:
			trailing = true
			r++

		case p[r] == '.' && p[r+1] == '/':
			// . element
			r += 2

		case p[r] == '.' && p[r+1] == '.' && (r+2 == n || p[r+2] == '/'):
			// .. element: remove to last /
			r += 3

			if w > 1 {
				// can backtrack
				w--

				if len(buf) == 0 {
					for w > 1 && p[w] != '/' {
						w--
					}
				} else {
					for w > 1 && buf[w] != '/' {
						w--
					}
				}
			}

		default:
			// Real path element.
			// Add slash if needed
			if w > 1 {
				bufApp(&buf, p, w, '/')
				w++
			}

			// Copy element
			for r < n && p[r] != '/' {
				bufApp(&buf, p, w, p[r])
				w++
				r++
			}
		}
	}

	// Re-append trailing slash
	if trailing && w > 1 {
		bufApp(&buf, p, w, '/')
		w++
	}

	// If the original string was not modified (or only shortened at the end),
	// return the respective substring of the original string.
	// Otherwise return a new string from the buffer.
	if len(buf) == 0 {
		return p[:w]
	}
	return string(buf[:w])
}

// Internal helper to lazily create a buffer if necessary.
// Calls to this function get inlined.
func bufApp(buf *[]byte, s string, w int, c byte) {
	b := *buf
	if len(b) == 0 {
		// No modification of the original string so far.
		// If the next character is the same as in the original string, we do
		// not yet have to allocate a buffer.
		if s[w] == c {
			return
		}

		// Otherwise use either the stack buffer, if it is large enough, or
		// allocate a new buffer on the heap, and copy all previous characters.
		length := len(s)
		if length > cap(b) {
			*buf = make([]byte, length)
		} else {
			*buf = (*buf)[:length]
		}
		b = *buf

		copy(b, s[:w])
	}
	b[w] = c
}
```

我阅读的时候主要就是使用go的调试工具来逐行查看运行效果的。主要就是多输入集中能想到的情况，尽可能让代码运行时覆盖掉所有路径。

这个函数最重要的就是w和r两个变量，r记录了处理到了输入路径的第几个字符，w用于标记当前为运行当前判断出来的在最后结果中的下标，这个下标可能是p的下标（如果buf为空），也可能是buf的下标（如果buf非空）。

还有比较重要的就是buf，buf用于存储最终会用到的路径，存储的内容可能比最终的真实结果长，但是因为程序运行结束前，w标记了应该存储到buf的第几个下标，最终返回的结果是buf[:w]

// 举一个需要注意的例子

// 如果输出参数是  "/tttttttttt/c/../d/../../aaa/yy"

那么程序运行过程中w和buf的值变化为：

```shell
w: 1, buf:
w: 2, buf:
w: 3, buf:
w: 4, buf:
w: 5, buf:
w: 6, buf:
w: 7, buf:
w: 8, buf:
w: 9, buf:
w: 10, buf:
w: 11, buf:
w: 12, buf:
w: 11, buf:
w: 12, buf: /tttttttttt/d  这里是 /tttttttttt/c/../d 得到的结果
w: 1, buf: /attttttttt/d 后面因为遇到了/aaa，因为前面的两个../，w不断回退，然后aaaa开始从w的位置也就是1开始覆盖buf
w: 2, buf: /aatttttttt/d
w: 3, buf: /aaattttttt/d
w: 4, buf: /aaa/tttttt/d
w: 5, buf: /aaa/yttttt/d
w: 6, buf: /aaa/yytttt/d w为6，虽然buf的长度不止6，但是w是标记哪些是真正计算出来的实际值的下标，返回的是用w截断的，返回的是buf[:w]
```

buf运行中之前写入的值可能会在之后被覆盖掉，最终运行的结果是以w和buf综合起来的，是buf在w处被截断的结果，为buf[:w]。