---
title: go的delve调试工具
date: 2023-02-17 11:00:00
categories:
- golang
---

起因是我在阅读一段比较难懂的源码，写了个单元测试想查看程序的具体运行状况，但是源码很长通过打印很难明白到底是如何执行的。由此想到可以用go的调试，但是我很久没用了，有些不记得了。于是再重新整理下。

## delve的安装

使用到调试器是Delve。这是专门为了go语言开发的调试工具。

安装

```shell
go get -u github.com/go-delve/delve/cmd/dlv
```

确认版本

```shell
dlv version
```

使用

切换到待调试的go项目所在的目录：

```shell
dlv debug
```

执行help命令之后可以看到有很多提示：

```shell
dlv debug 
Type 'help' for list of commands.
(dlv) help
The following commands are available:

Running the program:
    call ------------------------ Resumes process, injecting a function call (EXPERIMENTAL!!!)
    continue (alias: c) --------- Run until breakpoint or program termination.
    next (alias: n) ------------- Step over to next source line.
    rebuild --------------------- Rebuild the target executable and restarts it. It does not work if the executable was not built by delve.
    restart (alias: r) ---------- Restart process.
    step (alias: s) ------------- Single step through program.
    step-instruction (alias: si)  Single step a single cpu instruction.
    stepout (alias: so) --------- Step out of the current function.

Manipulating breakpoints:
    break (alias: b) ------- Sets a breakpoint.
    breakpoints (alias: bp)  Print out info for active breakpoints.
    clear ------------------ Deletes breakpoint.
    clearall --------------- Deletes multiple breakpoints.
    condition (alias: cond)  Set breakpoint condition.
    on --------------------- Executes a command when a breakpoint is hit.
    toggle ----------------- Toggles on or off a breakpoint.
    trace (alias: t) ------- Set tracepoint.
    watch ------------------ Set watchpoint.

Viewing program variables and memory:
    args ----------------- Print function arguments.
    display -------------- Print value of an expression every time the program stops.
    examinemem (alias: x)  Examine raw memory at the given address.
    locals --------------- Print local variables.
    print (alias: p) ----- Evaluate an expression.
    regs ----------------- Print contents of CPU registers.
    set ------------------ Changes the value of a variable.
    vars ----------------- Print package variables.
    whatis --------------- Prints type of an expression.

Listing and switching between threads and goroutines:
    goroutine (alias: gr) -- Shows or changes current goroutine
    goroutines (alias: grs)  List program goroutines.
    thread (alias: tr) ----- Switch to the specified thread.
    threads ---------------- Print out info for every traced thread.

Viewing the call stack and selecting frames:
    deferred --------- Executes command in the context of a deferred call.
    down ------------- Move the current frame down.
    frame ------------ Set the current frame, or execute command on a different frame.
    stack (alias: bt)  Print stack trace.
    up --------------- Move the current frame up.

Other commands:
    config --------------------- Changes configuration parameters.
    disassemble (alias: disass)  Disassembler.
    dump ----------------------- Creates a core dump from the current process state
    edit (alias: ed) ----------- Open where you are in $DELVE_EDITOR or $EDITOR
    exit (alias: quit | q) ----- Exit the debugger.
    funcs ---------------------- Print list of functions.
    help (alias: h) ------------ Prints the help message.
    libraries ------------------ List loaded dynamic libraries
    list (alias: ls | l) ------- Show source code.
    source --------------------- Executes a file containing a list of delve commands
    sources -------------------- Print list of source files.
    target --------------------- Manages child process debugging.
    transcript ----------------- Appends command output to a file.
    types ---------------------- Print list of types

Type help followed by a command for full documentation.
(dlv)
```

调试的过程中可以看到当前目录下会生成一个调试日志，日志名类似于__debug_bin528272488，不要操作这个文件，在执行quit命令推出delve的时候这个日志文件就会被自动删除。

## 断点

如果要调试程序，不可缺少的就是设置断点。所谓的断点，就是在调试程序时，程序指定到这个地方会停下，此时可以在dlv里查看当前的变量值是什么，或者可以之后逐行执行程序也就是每执行一行就停下来一次（因为默认情况下程序是会执行到下个断电或者执行到终端才停止的），或者可以设置接下来的条件断点（也就是满足指定条件时才触发断点）。

### 设置断点

#### **为入口main设置断点**

**这个不是必要操作，不操作这个也可以调试程序。**

（每个Go程序的入口是main.main，可以用break在这个入口设置断点）：

```shell
(dlv) break main.main
Breakpoint 1 set at 0x19aa9af for main.main() ./main.go:30
```

#### **在某一行设置断点**

如main.go的第10行：

```shell
(dlv) break main.go:10
Breakpoint 2 set at 0x10aea33 for main.main() ./main.go:10
```

> 如果设置断点时报错，比如：
>
> ```shell
> (dlv) break main.go:32
> Command failed: Location "main.go:32" ambiguous: /Users/xxxx/MyLocalFile/代码/fromweb/gopacket-fram/a_runner/main.go, /Users/rhettnina/go/pkg/mod/github.com/jinzhu/now@v1.1.5/main.go…
> ```
>
> 这是因为delve在设置断点时发生错误，即有多个文件包含相同的文件路径和行号，这可能是Go的模块的缓存导致的。可以在break后加上文件的完整路径来解决此问题：
>
> ```shell
> break /Users/xxxx/MyLocalFile/代码/fromweb/gopacket-fram/a_runner/main.go:32
> ```

#### **设置条件断点**

首先查看所有断点和断点ID：

```shell
(dlv) breakpoints
Breakpoint runtime-fatal-throw (enabled) at 0x103b500,0x103b600 for (multiple functions)() <multiple locations>:0 (0)
Breakpoint unrecovered-panic (enabled) at 0x103b9a0 for runtime.fatalpanic() /usr/local/Cellar/go/1.20.6/libexec/src/runtime/panic.go:1145 (0)
	print runtime.curg._panic.arg
Breakpoint 1 (enabled) at 0x19aa9e5 for main.main() ./main.go:32 (0)
```

我想设置的是main.go的第32行，

```shell
31	for i := 0; i < 3; i++ {
32		fmt.Println(i)
33	}
```

在i的值为2时出现断点。

执行如下命令：

```
condition 1 i==2
```

这里的1是断点ID，后面i==2是断点出现的条件。

### 查看断点

在调试模式下输入：

```shell
breakpoints
```

在不人为设置断点的默认情况下，会有一个panic函数断点：

```shell
(dlv) breakpoints
Breakpoint runtime-fatal-throw (enabled) at 0x103b600,0x103b500 for (multiple functions)() <multiple locations>:0 (0)
Breakpoint unrecovered-panic (enabled) at 0x103b9a0 for runtime.fatalpanic() /usr/local/Cellar/go/1.20.6/libexec/src/runtime/panic.go:1145 (0)
	print runtime.curg._panic.arg
```

## 调试

### 没有函数调用的情况

下面是一个示范，给一个例子进行调试：

```go
package main

import (
	"fmt"
)

func main() {
	nums := make([]int, 5)
	for i := 0; i < 5; i++ {
		nums[i] = i * i
	}
	fmt.Println(nums)
	for i := 0; i < 5; i++ {
		nums = append(nums, i)
	}
	fmt.Println(nums)
}
```

通过`dlv debug`进入调试模式

```shell
> dlv debug                                                       
Type 'help' for list of commands.
```

在main.go的第10行设置断点：

```shell
(dlv) break main.go:10
Breakpoint 1 set at 0x10b5c6f for main.main() ./main.go:10
```

执行c会在下一个断点处中止：

```shell
(dlv) c
> main.main() ./main.go:10 (hits goroutine(1):1 total:1) (PC: 0x10b5c6f)
     5:	)
     6:
     7:	func main() {
     8:		nums := make([]int, 5)
     9:		for i := 0; i < 5; i++ {
=>  10:			nums[i] = i * i
    11:		}
    12:		fmt.Println(nums)
    13:		for i := 0; i < 5; i++ {
    14:			nums = append(nums, i)
    15:		}
```

在此处可以查看全部包级变量：

```shell
(dlv) vars
......
sync.allPoolsMu = sync.Mutex {state: 0, sema: 0}
sync.allPools = []*sync.Pool len: 0, cap: 0, nil
sync.oldPools = []*sync.Pool len: 0, cap: 0, nil
```

因为vars输出特别多，可以用正则表达式过滤，比如过滤出变量名包含sync的所有变量：

```shell
(dlv) vars sync
runtime.asyncPreemptStack = 472
sync/atomic.firstStoreInProgress = 0
sync.expunged = (*interface {})(0xc000096020)
sync.allPoolsMu = sync.Mutex {state: 0, sema: 0}
sync.allPools = []*sync.Pool len: 0, cap: 0, nil
sync.oldPools = []*sync.Pool len: 0, cap: 0, nil
```

通过locals查看局部变量

```shell
(dlv) locals
nums = []int len: 5, cap: 5, [...]
i = 0
```

打印某个变量

```shell
(dlv) print nums
[]int len: 5, cap: 5, [0,0,0,0,0]
```

**此时nums还没有append第一个元素，虽然已经进入了循环内，因为调试模式此时的断点停止位置是在断点处也就是第10行执行之前停止的。**

然后继续执行c（即continue），让程序运行到下个断点：

```shell
(dlv) c
> main.main() ./main.go:10 (hits goroutine(1):2 total:2) (PC: 0x10b5c6f)
     5:	)
     6:
     7:	func main() {
     8:		nums := make([]int, 5)
     9:		for i := 0; i < 5; i++ {
=>  10:			nums[i] = i * i
    11:		}
    12:		fmt.Println(nums)
    13:		for i := 0; i < 5; i++ {
    14:			nums = append(nums, i)
    15:		}
```

除了执行c以外控制程序到下个断点处，还可以执行next即n，让调试程序逐行执行：

```shell
(dlv) n
> main.main() ./main.go:9 (PC: 0x10b5ca5)
     4:		"fmt"
     5:	)
     6:
     7:	func main() {
     8:		nums := make([]int, 5)
=>   9:		for i := 0; i < 5; i++ {
    10:			nums[i] = i * i
    11:		}
    12:		fmt.Println(nums)
    13:		for i := 0; i < 5; i++ {
    14:			nums = append(nums, i)
(dlv) n
> main.main() ./main.go:10 (hits goroutine(1):3 total:3) (PC: 0x10b5c6f)
     5:	)
     6:
     7:	func main() {
     8:		nums := make([]int, 5)
     9:		for i := 0; i < 5; i++ {
=>  10:			nums[i] = i * i
    11:		}
    12:		fmt.Println(nums)
    13:		for i := 0; i < 5; i++ {
    14:			nums = append(nums, i)
    15:		}
```

查看此时的本地变量，和预期相同：

```shell
(dlv) locals
nums = []int len: 5, cap: 5, [...]
i = 2
(dlv) print nums
[]int len: 5, cap: 5, [0,1,0,0,0]
(dlv) print i
2
```

此时i的值为2。下面设置一个条件断点，在第14行`	nums = append(nums, i)`当i为3时断点生效。

首先设置14行的断点：

```shell
(dlv)  break main.go:14
Breakpoint 2 set at 0x10b5d5a for main.main() ./main.go:14
```

通过输出可以看到该断点的ID是2。如果不设置条件，那么第14行就是一个断点，程序运行到第14行就会触发断点。下面设置这个断点的生效条件：

```shell]
(dlv) condition 2 i==3
```

这样只有i==3时才会触发断点2。

下面继续运行查看条件断点生效效果：

```shell
(dlv) c
> main.main() ./main.go:10 (hits goroutine(1):4 total:4) (PC: 0x10b5c6f)
     5:	)
     6:
     7:	func main() {
     8:		nums := make([]int, 5)
     9:		for i := 0; i < 5; i++ {
=>  10:			nums[i] = i * i
    11:		}
    12:		fmt.Println(nums)
    13:		for i := 0; i < 5; i++ {
    14:			nums = append(nums, i)
    15:		}
(dlv) c
> main.main() ./main.go:10 (hits goroutine(1):5 total:5) (PC: 0x10b5c6f)
     5:	)
     6:
     7:	func main() {
     8:		nums := make([]int, 5)
     9:		for i := 0; i < 5; i++ {
=>  10:			nums[i] = i * i
    11:		}
    12:		fmt.Println(nums)
    13:		for i := 0; i < 5; i++ {
    14:			nums = append(nums, i)
    15:		}
(dlv) c
[0 1 4 9 16]
> main.main() ./main.go:14 (hits goroutine(1):1 total:1) (PC: 0x10b5d5a)
     9:		for i := 0; i < 5; i++ {
    10:			nums[i] = i * i
    11:		}
    12:		fmt.Println(nums)
    13:		for i := 0; i < 5; i++ {
=>  14:			nums = append(nums, i)
    15:		}
    16:		fmt.Println(nums)
    17:	}
    18:
    19:	func cleanPath(p string) string {
(dlv) print i
3
```

可以看到触发14行的断点时，i的就是3，达到了满足断点触发的条件。

接下来继续运行c命令，程序推出：

````shell
(dlv) c
[0 1 4 9 16 0 1 2 3 4]
Process 77401 has exited with status 0
````

### 查看有函数调用的情况

#### 在函数内部设置断点

**想要查看函数内部的运行情况，可以在函数内部设置断点。**

以下面这个源代码的调试为例：

```shell
package main

import (
	"fmt"
)

func main() {
	i := 10
	j := 20
	r := multi(i, j)
	fmt.Println(r)
}

func multi(i, j int) int {
	r := i * j
	return r
}
```

现在进入multi函数前设置断点：

```shell
dlv debug
Type 'help' for list of commands.
(dlv) break main.go:9
Breakpoint 1 set at 0x10b5c21 for main.main() ./main.go:9
(dlv) c
> main.main() ./main.go:9 (hits goroutine(1):1 total:1) (PC: 0x10b5c21)
Warning: listing may not match stale executable
     4:		"fmt"
     5:	)
     6:
     7:	func main() {
     8:		i := 10
=>   9:		j := 20
    10:		r := multii(i, j)
    11:		fmt.Println(r)
    12:	}
    13:
    14:	func multii(i, j int) int {
```

下面单步执行

```shell
(dlv) n
> main.main() ./main.go:10 (PC: 0x10b5c2a)
Warning: listing may not match stale executable
     5:	)
     6:
     7:	func main() {
     8:		i := 10
     9:		j := 20
=>  10:		r := multi(i, j)
    11:		fmt.Println(r)
    12:	}
    13:
    14:	func multi(i, j int) int {
    15:		r := i * j
(dlv) n
> main.main() ./main.go:11 (PC: 0x10b5c3e)
Warning: listing may not match stale executable
     6:
     7:	func main() {
     8:		i := 10
     9:		j := 20
    10:		r := multi(i, j)
=>  11:		fmt.Println(r)
    12:	}
    13:
    14:	func multi(i, j int) int {
    15:		r := i * j
    16:		return r
```

发现跳过了进入函数的内部。下面重新执行以下，在函数内部设置断点：

```shell
> dlv debug
(dlv) break main.go:9
Breakpoint 1 set at 0x10b5c21 for main.main() ./main.go:9
(dlv) c
> main.main() ./main.go:9 (hits goroutine(1):1 total:1) (PC: 0x10b5c21)
     4:		"fmt"
     5:	)
     6:
     7:	func main() {
     8:		i := 10
=>   9:		j := 20
    10:		r := multi(i, j)
    11:		fmt.Println(r)
    12:	}
    13:
    14:	func multi(i, j int) int {
(dlv) break main.go:15
Breakpoint 2 set at 0x10b5d00 for main.multi() ./main.go:15
```

下面继续执行：

```shell
(dlv) c
> main.main() ./main.go:9 (hits goroutine(1):1 total:1) (PC: 0x10b5c21)
     4:		"fmt"
     5:	)
     6:
     7:	func main() {
     8:		i := 10
=>   9:		j := 20
    10:		r := multi(i, j)
    11:		fmt.Println(r)
    12:	}
    13:
    14:	func multi(i, j int) int {
(dlv) break main.go:15
Breakpoint 2 set at 0x10b5d00 for main.multi() ./main.go:15
(dlv) c
> main.multi() ./main.go:15 (hits goroutine(1):1 total:1) (PC: 0x10b5d00)
    10:		r := multi(i, j)
    11:		fmt.Println(r)
    12:	}
    13:
    14:	func multi(i, j int) int {
=>  15:		r := i * j
    16:
```

可以看到此时到达了函数内部。

在函数内部，有不同的命令可以用，args可以查看函数接收的参数是什么

```shell
(dlv) args
i = 10
j = 20
~r0 = 0
```

stack查看调用栈：

```shell
(dlv) stack
0  0x00000000010b5d00 in main.multi
   at ./main.go:15
1  0x00000000010b5c39 in main.main
   at ./main.go:10
2  0x00000000010394f3 in runtime.main
   at /usr/local/Cellar/go/1.20.6/libexec/src/runtime/proc.go:250
3  0x0000000001066b61 in runtime.goexit
   at /usr/local/Cellar/go/1.20.6/libexec/src/runtime/asm_amd64.s:1598
```

查看goroutine相关信息：

```shell
(dlv) goroutines
* Goroutine 1 - User: ./main.go:15 main.multi (0x10b5d00) (thread 6973579)
  Goroutine 2 - User: /usr/local/Cellar/go/1.20.6/libexec/src/runtime/proc.go:382 runtime.gopark (0x103999d) [force gc (idle)]
  Goroutine 3 - User: /usr/local/Cellar/go/1.20.6/libexec/src/runtime/proc.go:382 runtime.gopark (0x103999d) [GC sweep wait]
  Goroutine 4 - User: /usr/local/Cellar/go/1.20.6/libexec/src/runtime/proc.go:382 runtime.gopark (0x103999d) [GC scavenge wait]
  Goroutine 5 - User: /usr/local/Cellar/go/1.20.6/libexec/src/runtime/proc.go:382 runtime.gopark (0x103999d) [finalizer wait]
[5 goroutines]
(dlv) goroutine
Thread 6973579 at ./main.go:15
Goroutine 1:
	Runtime: ./main.go:15 main.multi (0x10b5d00)
	User: ./main.go:15 main.multi (0x10b5d00)
	Go: <autogenerated>:1 runtime.newproc (0x1069365)
	Start: /usr/local/Cellar/go/1.20.6/libexec/src/runtime/proc.go:145 runtime.main (0x1039320)
```

最后通过quit退出函数。

#### 设置函数名处断点

比如对于上面的源码，设置断点：

```shell
break main
```

那么执行c命令时会在调用该函数处触发断点，接下来使用n即可进入函数内部查看具体调用情况。

此外还有很多其他可以运行的命令，在help命令里列出来了，这里介绍了一些基本的常用的命令。

print命令可以用p代替，表示缩写。

#### 调试过程中执行函数调用

这个步骤就是在执行过程中自定义参数调用函数，比如对于上面的例子程序：

```shell
dlv debug                                           
Type 'help' for list of commands.
(dlv) break main.main
Breakpoint 1 set at 0x10b5c0a for main.main() ./main.go:7
(dlv) break main.go:10
Breakpoint 2 set at 0x10b5c2a for main.main() ./main.go:10
(dlv) c
> main.main() ./main.go:7 (hits goroutine(1):1 total:1) (PC: 0x10b5c0a)
     2:
     3:	import (
     4:		"fmt"
     5:	)
     6:
=>   7:	func main() {
     8:		i := 10
     9:		j := 20
    10:		r := multi(i, j)
    11:		fmt.Println(r)
    12:	}
(dlv) c
> main.main() ./main.go:10 (hits goroutine(1):1 total:1) (PC: 0x10b5c2a)
     5:	)
     6:
     7:	func main() {
     8:		i := 10
     9:		j := 20
=>  10:		r := multi(i, j)
    11:		fmt.Println(r)
    12:	}
    13:
    14:	func multi(i, j int) int {
    15:		r := i * j
```

此时到达了断点，然后按行执行下一步：

```shell
(dlv) n
> main.main() ./main.go:11 (PC: 0x10b5c3e)
     6:
     7:	func main() {
     8:		i := 10
     9:		j := 20
    10:		r := multi(i, j)
=>  11:		fmt.Println(r)
    12:	}
    13:
    14:	func multi(i, j int) int {
    15:		r := i * j
    16:		return r
```

程序里调用multi传递的参数值是10和20，如果在调试中我突然想测试一些极端情况，可以执行call命令，自定义参数调用函数，比如传递-1 -1给multi：

````shell
(dlv) call multi(-1,-1)
> main.main() ./main.go:11 (PC: 0x10b5c3e)
Values returned:
	~r0: 1

     6:
     7:	func main() {
     8:		i := 10
     9:		j := 20
    10:		r := multi(i, j)
=>  11:		fmt.Println(r)
    12:	}
    13:
    14:	func multi(i, j int) int {
    15:		r := i * j
    16:		return r
````

可以看到有值返回了，返回的值是1，也就是multi(-1,-1)的结果

```shell
Values returned:
	~r0: 1
```

因为上一步程序调试停在了11行fmt.Println(r)，这里中途执行call不会影响上一步调试的停止，中途执行call之后程序还是停在了11行，下面把程序执行完毕：

```shell
(dlv) n
200
> main.main() ./main.go:12 (PC: 0x10b5cba)
     7:	func main() {
     8:		i := 10
     9:		j := 20
    10:		r := multi(i, j)
    11:		fmt.Println(r)
=>  12:	}
    13:
    14:	func multi(i, j int) int {
    15:		r := i * j
    16:		return r
    17:	}
(dlv) c
Process 11724 has exited with status 0
```
