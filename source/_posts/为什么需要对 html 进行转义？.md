---
title: 为什么需要对 html 进行转义？
date: 2021-3-9 00:00:00
tags:
- 原创
categories:
- 其他
---

在 html 中，有一些符号<>等等都是有特定含义的，比如<>是一个标签的开始和结束。如果在一个.html文件写<>，会被解析为标签。那么如果想在页面显示这些特定字符呢？那么就需要对这些字符进行转义，然后写到 html 中，就不会被浏览器认为是 html 中的特殊字符。

## go 中的方法

在 go 中，可以使用官方 html 包的 EscapeString 方法，将一个字符串中的在 html 中有特定含义的字符转义，官方对于这个方法的注释如下：

> EscapeString escapes special characters like "<" to become "&lt;". It escapes only five such characters: <, >, &, ' and ". UnescapeString(EscapeString(s)) == s always holds, but the converse isn't always true.

官方示例：

Code:[play](https://godoc.org/html?play=EscapeString)

```go
const s = `"Fran & Freddie's Diner" <tasty@example.com>`
fmt.Println(html.EscapeString(s))
```

Output:

```text
&#34;Fran &amp; Freddie&#39;s Diner&#34; &lt;tasty@example.com&gt;
```

与 EscapeString 对应的方法叫 [UnescapeString](https://golang.org/src/html/escape.go#L187)，官方注释如下：

> UnescapeString unescapes entities like "&lt;" to become "<". It unescapes a larger range of entities than EscapeString escapes. For example, "&aacute;" unescapes to "á", as does "&#225;" and "&#xE1;". UnescapeString(EscapeString(s)) == s always holds, but the converse isn't always true.

官方示例：

Code:[play](https://godoc.org/html?play=UnescapeString)

```go
const s = `&quot;Fran &amp; Freddie&#39;s Diner&quot; &lt;tasty@example.com&gt;`
fmt.Println(html.UnescapeString(s))
```

Output:

```shell
"Fran & Freddie's Diner" <tasty@example.com>
```

## 我的测试

如果一个文件 data.html 中的内容如下：

```html
<p>请在输入框内贴入你需要转换的HTML代码</p>
```

那么用chrome打开这个文件显示结果为：

![QQ20201115-094330@2x.png](http://ww1.sinaimg.cn/large/006gLprLly1gkpm4c93o3j30n2030jro.jpg)

如果想在浏览器显示< 呢？

用 go 的EscapeString方法或者[在线html转义工具](https://www.sojson.com/rehtml)对<进行转义，得到`&lt;`，将`&lt;`加到段落最后，就不会被认为是一个标签的开始了。data.html 改成这样：

```html
<p>请在输入框内贴入你需要转换的HTML代码&lt;</p>
```

chrome 的显示效果为：

![QQ20201115-094550@2x.png](http://ww1.sinaimg.cn/large/006gLprLly1gkpm6o38b5j30l001y3yr.jpg)
