---
title: 北极星configMao更新
date: 2023-11-02 11:00:00
categories:
- 运维
---
## 北极星简介

参考[官方文档](https://polarismesh.cn/docs/%E5%8C%97%E6%9E%81%E6%98%9F%E6%98%AF%E4%BB%80%E4%B9%88/%E7%AE%80%E4%BB%8B/)

北极星是腾讯开源的服务治理平台。

北极星致力于打造一个支持多语言、多框架的服务治理平台，帮助用户解决分布式服务或者微服务架构中的服务管理、流量管理、配置管理、故障容错和可观测性问题。

北极星具备服务管理、流量管理、故障容错、配置管理和可观测性五大功能

## ConfigMap

ConfigMap 是一种 API 对象，用来将非机密性的数据保存到键值对中。

ConfigMap 是一种用于存储配置数据的 API 资源。它允许你将配置信息从应用程序中分离出来，使得配置能够在不重新构建镜像或重启 Pod 的情况下进行更新。类似于实现配置的热更新。

configMap是一个文件，格式参考：

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  database-url: "mysql://username:password@mysql-server:3306/mydatabase"
  api-key: "abcdef123456"
  max-connections: "10"
```

### 读取

ConfigMap 可以通过卷挂载到 Pod 的容器内，使得容器可以读取其中的配置信息。

程序可以通过不同的方式访问 ConfigMap 中的配置信息：ConfigMap 的数据可以通过环境变量注入到容器内，ConfigMap 的数据可以通过卷挂载到容器内的文件系统。程序可以直接读取这些文件来获取配置信息。或者程序可以直接从 Kubernetes API 中获取 ConfigMap 的数据。

### 写入

有几种常见的方式可以更新 ConfigMap 中的配置信息：

**手动更新：** 你可以直接通过编辑 ConfigMap 对象的 YAML 文件或使用命令行工具（如 `kubectl edit configmap`）手动更新 ConfigMap 的数据。

**通过命令行工具 kubectl replace：** 使用 `kubectl replace` 命令可以用新的 ConfigMap 替换旧的 ConfigMap。这个命令会将整个 ConfigMap 替换为新的内容。

**通过命令行工具 kubectl apply**

**通过 API 更新：** 通过 Kubernetes 的 API，可以使用客户端库或直接发送 HTTP 请求来更新 ConfigMap。

**使用 Helm：** Helm 是 Kubernetes 的一个包管理工具，可以用于部署和管理应用程序。通过 Helm Charts，你可以定义配置文件，并在需要更新时通过 Helm 更新部署。

