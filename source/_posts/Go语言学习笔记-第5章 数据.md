---
title: Go语言学习笔记-第5章 数据
date: 2020-10-4 00:00:00
tags:
- 读书
categories:
- golang
---

### 字段标签

字==结构体后面的struct字段里面的反引号``括起来的内容就是字段标签，
如下：

```
type user struct{
    name string `名字`
    sex bool `性别`
}
func main(){
   u:=user{"Tom",1}
   v:=reflect.ValueOf(u)
   t:=v.Type()
   for i,n:=0,t.NumField();i<n;i++{
   fmt.Println("%s:%v\n",t.Field(i).Tag,v.Field(i))
   }
}
```

输出结果为:

```
姓名:Tom
性别:1
```

综上所述，获取结构体值采用的方法是**reflect.ValueOf()**
获取字段标签的方法是**reflect.ValueOf().Type()**

### 内存布局  主要利用unsafe包检查内存情况

golang定义的int类型会根据不同的平台切换成int32或者int64,int32和uint32都是占四个字节。而int64和uint64都是占8个字节 **byte和bool占一个字节**

结构体的内存总是一次性分配的，各个字段在相邻的地址空间内按顺序排列。对于引用类型，字符串和数组，结构体中只包含其基本(头部)数据,所有匿名字段成员也被包含在内。

unsafe.Alignof返回的是结构体中基础类型的大小，也就是对齐宽度
unsafe.Sizeof() 返回对象的大小。以字节为单位
unsafe.Pointer(&v) 返回v的地址（也就是v的首地址),可以与unsafe.Sizeof()相加，获得末地址

**输出的时候可以使用%p占指针的位，可以使用%d占数值的位**

**分配内存的时候，字段需要做对齐处理，也就是以字段中最长的基础类型宽度为标准**
