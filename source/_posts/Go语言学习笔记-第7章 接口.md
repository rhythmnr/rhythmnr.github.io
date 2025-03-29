---
title: Go语言学习笔记-第7章 接口
date: 2020-10-6 00:00:00
tags:
- 读书
categories:
- golang
---

## 第七章 接口

来源：《Go 语言学习笔记》

### 7.1 定义

是一组方法声明的集合。接口运行的时候回有运行期开销。

接口最常见的使用场景，是对包外提供访问，或者预留空间。

也可以先写方法然后抽象出接口。  
接口可以嵌入其他类型接口，那么需要实现这些接口里的所有方法才可以。
接口名字通常以er结尾。

结构体的方法实现接口的时候，结构体的方法不能包含指针。

实现方法一般写成func (db *Database)funcname(){}这样的形式

对于一个方法，func(*data)和func(data)，只有形如t=&data的形式才可以调用者两个方法，t=data是不可以的。

```
type tester interface{
    test()
    string() string
}
type data()
func(*data)test()
func(data)stirng(string){return "123"}
func main(){
    var d data
    // var t tester=d   FIXME: 这个是不对的，因为data类型没有实现test和string,*data实现了
    var t tester=&d
    t.test()
    fmt.Println(t.string()
}
```

**空接口可以被赋值为任何对象,默认值是nil**
也可以在结构体里面写接口

-----------

# 7.1 定义

1. 接口变量的默认值是nil
2. 接口之间是可以进行是否相等的判断的
3. 可以接口套接口
4. 空接口可以实现任何接口，比如一个空接口叫test,然后另一个接口叫data，那么可以定义 var d data=&test **空接口可以被赋值为任何类型的对象**
5. 如果接口A实现了接口B的所有方法，那么可以直接赋值为 var b B=&A，然后可以通过b调用A实现的具体方法了
6. **Go不是一种典型的OO语言，它在语法上不支持类和继承的概念。**
    所以接口和结构体非常重要

[下面来自掘金](https://juejin.im/post/5a6873fd518825734501b3c5)

~ 补充：*方法和函数的区别*：方法一般是指某个结构体的方法，类似于 func (a *struct)function(args){}，而函数就是单纯的接收一个输入然后给出一个输出 func funcname(in)out{}.也就是说方法是作用于某个数据类型上的函数。
7. Go语言支持的除Interface类型外的任何其它数据类型都可以定义其method（而并非只有struct才支持method），只不过实际项目中，method(s)多定义在struct上而已。
8. **只要你实现了我，你就可以调我的方法**
也就说只要你实现了我，就可以把你看成我。

```
   type MyInterface interface{
       Print()
   } # 接口MyInterface定义，实现这个接口需要Print()方法实现才可以
   
   func TestFunc(x MyInterface) {}
# MyInterface的方法
   type MyStruct struct {}
# MyStruct接口定义
   func (me MyStruct) Print() {}
# MyStruct的Print方法，实现了MyInterface接口
   func main() {
       var me MyStruct
       # me是一个MyStruct接口，实现了MYInterface接口
       TestFunc(me)
       # 因为MyStruct实现了MyInterface接口，所以可以其调用MyInterface的方法
   }
```

例子

```
    package sort

    // A type, typically a collection, that satisfies sort.Interface can be
    // sorted by the routines in this package.  The methods require that the
    // elements of the collection be enumerated by an integer index.
    type Interface interface {
        // Len is the number of elements in the collection.
        Len() int
        // Less reports whether the element with
        // index i should sort before the element with index j.
        Less(i, j int) bool
        // Swap swaps the elements with indexes i and j.
        Swap(i, j int)
    }
    
    ...
    
    // Sort sorts data.
    // It makes one call to data.Len to determine n, and O(n*log(n)) calls to
    // data.Less and data.Swap. The sort is not guaranteed to be stable.
    func Sort(data Interface) {
        // Switch to heapsort if depth of 2*ceil(lg(n+1)) is reached.
        n := data.Len()
        maxDepth := 0
        for i := n; i > 0; i >>= 1 {
            maxDepth++
        }
        maxDepth *= 2
        quickSort(data, 0, n, maxDepth)
    }
    
```

调用

```
   type Person struct {
   Name string
   Age  int
   }
   
   func (p Person) String() string {
       return fmt.Sprintf("%s: %d", p.Name, p.Age)
   }
   
   // ByAge implements sort.Interface for []Person based on
   // the Age field.
   type ByAge []Person //自定义
   
   func (a ByAge) Len() int           { return len(a) }
   func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
   func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }
   
   func main() {
       people := []Person{
           {"Bob", 31},
           {"John", 42},
           {"Michael", 17},
           {"Jenny", 26},
       }
   
       fmt.Println(people)
       sort.Sort(ByAge(people))
       fmt.Println(people)
   }
   
```

因为ByAge实现了Interface的方法，所以可以直接写ByAge(people)，把这个当做Interface传给Sort

## hiding implementation detail （隐藏具体实现）

是一些源码，读不下去了

## interface的内存布局

**当你实现了我，也可以把你强制转换为我**

```
    type Stringer interface {
        String() string
    }
    
    type Binary uint64
    
    func (i Binary) String() string {
        return strconv.Uitob64(i.Get(), 2)
    }
    
    func (i Binary) Get() uint64 {
        return uint64(i)
    }
    
    func main() {
        b := Binary{}
        s := Stringer(b)
        fmt.Print(s.String())
    }
```
