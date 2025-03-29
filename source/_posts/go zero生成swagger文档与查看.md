---
title: go zero生成swagger文档与查看
date: 2022-11-24 20:00:00
categories:
- 运维
---
## 生成json格式的API文档

```
goctl api plugin -plugin goctl-swagger="swagger -filename example.json" -api example.api -dir .
```

## 本地查看生成的swagger文档

启动swagger editor编辑器

```shell
git clone https://github.com/swagger-api/swagger-editor.git
cd swagger-editor
npm install
npm run build
npm start
```

本地启动swagger ui，查看swagger文档

```
git clone https://github.com/swagger-api/swagger-ui.git
cd swagger-ui
npm run dev
```

将json或者yml格式的swagger文档比如example.json复制到dev-helpers目录下

将dev-helpers/dev-helper-initializer.js文件的

```
url: "https://petstore.swagger.io/v2/swagger.json",
```

替换为

```
url: "./example.json",
```

打开http://localhost:3200/即可看到效果。

## PS: 本地开启swagger editor

```
git clone git@github.com:swagger-api/swagger-editor.git
cd swagger-editor
npm start
```

可以在http://localhost:3001/编辑swagger文档

参考

[Setting up a dev environment](https://github.com/swagger-api/swagger-ui/blob/master/docs/development/setting-up.md)

