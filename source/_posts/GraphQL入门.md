---
title: GraphQL入门
date: 2023-6-11 00:00:00
categories:
- 其他
---

# GraphQL入门

GraphQL 是一个用于 API 的查询语言

GraphQL 是一种针对 Graph（图状数据）进行查询特别有优势的 Query Language（查询语言），所以叫做 GraphQL。

GraphQL 跟用作存储的 NoSQL 等没有必然联系，**GraphQL 背后的实际存储可以选择 NoSQL 型或是 SQL 类型的数据库，甚至任意其它存储方式（例如文本文件、存内存里等等）**。虽然GraphQL是查询语言，但它与数据库没有直接关系

一旦一个 **GraphQL 服务运行起来（**通常在 web 服务的一个 URL 上），它就能接收 GraphQL 查询，并验证和执行。

GraphQL 是 Facebook 发明的

并不是所有场景都适用于 GraphQL 的，有些很简单的事情就应该用 RESTful API 来实现

GraphQL 尽管查询的数据是图状数据结构，但实际获得的数据视图是树状数据结构。每一个 GraphQL 查询或更新都有自己的根节点，然后所有的数据都是从根结点展开出去的。

GraphQL 对你的 API 中的数据提供一套易于理解的完整描述，使得客户端能够准确地获得它需要的数据，而且没有任何冗余，也让 API 更容易地随着时间推移而演进，还能用于构建强大的开发者工具。



其他乱七八糟的整理

------

**brew update和brew upgrade有什么区别**

`brew update` 和 `brew upgrade` 是两个常用于管理 macOS 上 Homebrew 软件包管理器的命令，它们的区别如下：

1. `brew update`：用于更新 Homebrew 自身以及其软件包清单。执行 `brew update` 会从 Homebrew 的 Git 仓库中拉取最新的版本信息，包括软件包清单、Formula（软件包的构建脚本）等，并更新本地的 Homebrew 安装。这样可以确保使用最新的软件包清单来安装和更新软件包。
2. `brew upgrade`：用于升级已安装的软件包到最新版本。执行 `brew upgrade` 会检查已安装的软件包，如果有新版本可用，则会升级到最新版本。需要注意的是，`brew upgrade` 只会升级已安装的软件包，不会安装新的软件包。

简而言之，`brew update` 是更新 Homebrew 本身和软件包清单，而 `brew upgrade` 是升级已安装的软件包到最新版本。通常情况下，先执行 `brew update` 更新软件包清单，然后再执行 `brew upgrade` 来升级已安装的软件包。



下面是一个简单的使用 `brew update` 和 `brew upgrade` 的例子：

1. 执行 `brew update` 更新 Homebrew 自身和软件包清单：

```
rubyCopy code
$ brew update
```

这会从 Homebrew 的 Git 仓库中拉取最新的版本信息，并更新本地的 Homebrew 安装。

1. 执行 `brew upgrade` 升级已安装的软件包到最新版本：

```
rubyCopy code
$ brew upgrade
```

这会检查已安装的软件包，如果有新版本可用，则会升级到最新版本。

需要注意的是，`brew update` 和 `brew upgrade` 只会更新和升级 Homebrew 管理的软件包，不会影响系统自带的软件包或其他方式安装的软件包。在执行 `brew update` 和 `brew upgrade` 时，可以根据需要选择性更新和升级特定的软件包，如：

- `brew update <formula>`：更新指定的软件包。
- `brew upgrade <formula>`：升级指定的软件包。

以上是使用 `brew update` 和 `brew upgrade` 的简单示例，具体的使用方法和参数可以参考 Homebrew 的官方文档或命令行帮助。
