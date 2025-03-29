---
title: struct转为map，并使得未被赋值的字段不出现在map里
date: 2022-9-25 00:00:00
tags:
- 原创
categories:
- golang
---

## 问题

今天写代码遇到了一个函数实现的问题，该函数的功能是根据某个struct的json tag，构建对应的map[string]interface{}。如果struct中某个字段未赋值或者为默认值，那么得到的map中不要出现该字段。比如

```go
type Student struct {
	Name string `json:"name"`
	Age  int    `json:"age"`
}
```

`s1:=Student{"Alice", 12}`得到的结果是`map[string]interface{}{"name":"Alice", "age":12}`

`s2:=Student{Name: "Bob"}`得到的结果是`map[string]interface{}{"name":"Bob"}`

## 解决方法

```go
type StructTagDecoder struct {
	tag string
}

func (d *StructTagDecoder) Decode(i interface{}) map[string]interface{} {
	v := reflect.ValueOf(i)
	if v.Kind() == reflect.Ptr {
		v = v.Elem()
	}
	if v.Kind() != reflect.Struct {
		panic(fmt.Errorf("toMap only accepts structs, got (%T)", v))
	}
	typ := v.Type()
	out := make(map[string]interface{})
	for i := 0; i < v.NumField(); i++ {
		fi := typ.Field(i)
		if tagv := fi.Tag.Get(d.tag); tagv != "" {
			field := v.Field(i).Interface()
			if field != reflect.Zero(reflect.TypeOf(field)).Interface() {
				out[tagv] = v.Field(i).Interface()
			}
		}
	}
	return out
}
```

```go
type Student struct {
	Name string `json:"name"`
	Age  int    `json:"age"`
}

func main() {
	s1 := Student{"Alice", 12}
	s2 := Student{Name: "Bob"}
	s3 := Student{"Charles", 0}
	decoder := StructTagDecoder{tag: "json"}
	fmt.Println(decoder.Decode(s1))
	fmt.Println(decoder.Decode(s2))
	fmt.Println(decoder.Decode(s3))
}
// 运行结果如下
// map[age:12 name:Alice]
// map[name:Bob]
// map[name:Charles]
```

## 分析

```
func (d *StructTagDecoder) Decode(i interface{}) map[string]interface{} {
	v := reflect.ValueOf(i)
	if v.Kind() == reflect.Ptr {
		v = v.Elem()
	}
	if v.Kind() != reflect.Struct {
		panic(fmt.Errorf("toMap only accepts structs, got (%T)", v))
	}
	typ := v.Type()
	out := make(map[string]interface{})
	for i := 0; i < v.NumField(); i++ {
		fi := typ.Field(i)
		if tagv := fi.Tag.Get(d.tag); tagv != "" {
			field := v.Field(i).Interface()
			if field != reflect.Zero(reflect.TypeOf(field)).Interface() {
				out[tagv] = v.Field(i).Interface()
			}
		}
	}
	return out
}
```

`reflect.ValueOf(i)`返回存储于interface{}中的具体值的新的值，ValueOf()返回的是一个Value struct

判断`if v.Kind() == reflect.Ptr`，如果原始值是一个指针类型的值，那么获取它的原始值。

获取类型`	typ := v.Type()`

v.NumField()获取结构体字段数量，遍历结构体的各个字段：

fi := typ.Field(i)获取结构体的第i个字段，`fi.Tag.Get(d.tag)`来获取json tag，获取成功后获取该字段的值`field := v.Field(i).Interface()`，这里把这个值转换为了interface，为了兼容。

下面需要判断这个值是否是默认值：

reflect.TypeOf(field)获取代表field动态类型的反射类型

> TypeOf returns the reflection Type that represents the dynamic type of i.

reflect.Zero(reflect.TypeOf(field))返回反射类型reflect.TypeOf(field)的零值

> // Zero returns a Value representing the zero value for the specified type.
>
> // The result is different from the zero value of the Value struct,
>
> // which represents no value at all.
>
> // For example, Zero(TypeOf(42)) returns a Value with Kind Int and value 0.
>
> // The returned value is neither addressable nor settable.

如果field != reflect.Zero(reflect.TypeOf(field)).Interface()，则认为field不是零值，则设置map。

参考

[Quick way to detect empty values via reflection in Go](https://stackoverflow.com/questions/13901819/quick-way-to-detect-empty-values-via-reflection-in-go)

