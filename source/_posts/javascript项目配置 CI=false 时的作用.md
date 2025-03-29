---
title: javascript项目配置 CI=false 时的作用
date: 2025-1-10 00:00:00
tags:
- javascript
---

背景：

开发的一个前端项目，在本地执行npm build没有问题，打印了一些eslint的warning。但是在github runner上报错了，报错内容为所有的warning。

其实报错的都是eslint的一些警告，应该是github runner上所谓类似等级的东西比较高，所以把eslint的警告视为错误。而我的本地等级比较低，所以不会把eslint的警告视为错误

对于这种场景，需要修改一个配置，修改为CI=false，参考修改如下：

```shell
    "build:dev": "cross-env REACT_APP_APP_ENV=dev react-app-rewired build CI=false",
    "build:qa": "cross-env REACT_APP_APP_ENV=qa react-app-rewired build CI=false",
    "build:prod": "cross-env REACT_APP_APP_ENV=prod react-app-rewired build CI=false",
```

`CI` 是一个常见的环境变量，代表 "Continuous Integration"（持续集成）。在一些持续集成环境中（如Jenkins、Travis CI等），`CI` 环境变量通常被设置为 `true`，以便在**严格模式**下运行，比如在测试失败时阻止构建的继续。严格模式下如果有eslint warning就会直接程序exit。

一些工具和库在检测到 `CI=true` 时会执行额外的检查和验证。例如，某些测试框架可能会在CI环境中强制所有测试通过，而不会允许任何警告或错误。某些构建工具和脚本在CI环境中可能会有特定行为。例如，Create React App（CRA）在CI环境中会将构建的所有警告视为错误，导致构建失败。

在这个脚本中，`CI=false` 是将 `CI` 环境变量设置为 `false`，通常用于在本地开发或非CI环境中运行构建过程。这可能会禁用某些CI特有的行为，比如严格的错误检查。

设置 `CI=false` 环境变量，以确保在构建过程中不会触发某些CI特有的行为。

这对于在本地开发环境中进行构建特别有用，因为你可能不希望在本地构建时触发。
