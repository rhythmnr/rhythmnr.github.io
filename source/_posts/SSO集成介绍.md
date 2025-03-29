---
title: SSO集成介绍
date: 2024-12-1 10:30:00
categories:
- mobile
---

## 简介

SSO是一个用户验证方式，比如可以通过微软的Azure来实现。企业可以使用微软的Azure来管理内部员工的登录。

## 使用SSO的流程

主要流程如下：

1. 企业管理员将企业内的各个系统的登录方式都配置为Azure SSO
2. 当有新员工入职时，企业管理员将员工信息录入Azure，主要会录入用户名、初始密码等
3. 新员工登录企业内**管理系统**，修改密码（企业内管理系统在处理修改密码请求时，会请求Azure的API，修改Azure里用户的密码）
4. 新员工登录企业内的其他**业务系统**，业务系统会在登录阶段跳转到微软的Azure SSO登陆页面，用户在该页面输入用户名和密码（Azure系统检测到用户名和密码正确后，会回传一个token给业务系统，业务系统拿到该token后，使用该token请求Azure的API，获取用户信息，比如用户名、用户所在部门等），用户名和密码输入成功后，会跳回业务系统页面，用户进入业务系统正常操作

综上，用户的用户名和密码会存储在Azure的数据库中，而不是公司的数据库中

## 配置应用使用SSO

因为开发的app应用需要和公司的其他应用一样，需要配置为SSO登陆。下面介绍上面流程的第一步，如何配置应用的登陆方式为Azure SSO。

主要参数如下：

```shell
// 当前域名
LOGIN_REDIRECT=https://xxx.com
// 重定向地址
OATH2_REDIRECT_URL=https://xxx.com/user/login_callback
// client_id，在app的详情中查看，由管理员给的
OATH2_CLIENT_ID=aaaaa-aaaa-aaaa-baaa-aaaaaa
// tent_id，密钥，管理员给的
OAUTH_TENANT_ID=42xxx-f
// 在Azure上生成的，验证机器是否允许登陆
OATH2_CLIENT_SECRET=mv~fffr
```

上面的这些参数都是在微软的Azure后台配置的。

app应用一般会使用一些SDK来集成SSO，SDK中需要配置的参数也差不多为上述参数。

最终效果为：当用户点击登陆，即调用**业务系统**的登录接口时，业务系统会马上调用到Azure的服务去登陆，这是一个重定向的动作。这时候浏览器会跳出Microsoft的登陆

> 比如flutter中，可以使用aad_oauth SDK来实现SSO，主要就是调用如下两个函数：
>
> ```dart
>     final config = Config(
>       navigatorKey: "xxx",
>       tenant: "xxx",
>       clientId: "xxx",
>       redirectUri: "xxx",
>       scope: "xxx",
>     );
>     _oauth = AadOAuth(config);
> 		await oauth.login();
>     String? accessToken = await oauth.getAccessToken();
> ```
>
> 第二个返回的accessToken就是传给业务系统的token
>
> 总之oauth.login()和oauth.getAccessToken很神奇，他们的执行逻辑大概是：会自动使用之前在该设备上创建的accessToken，如果accessToken过期会先尝试申请新的（应该是用户名和密码都没改的情况下才能成功），如果申请新的失败会让用户重新登陆

## OAuth2标准

微软的Azure使用的就是OAuth2标准，这是关于授权的开放网络标准，在全世界得到广泛应用。

OAuth 就是一种授权机制。数据的所有者告诉系统，同意授权第三方应用（比如上述的**业务系统**）进入系统（即**微软的Azure系统**），获取这些数据。系统从而产生一个短期的进入令牌（token），用来代替密码，供第三方应用使用。

OAuth2中有几个主要概念，

Third-party application：第三方应用程序，如上述的业务系统

Resource Owner：资源所有者，如上述的员工，员工拥有自己的个人信息资源

Authorization server：认证服务器，即专门用来处理认证的服务器，如上述的微软Azure服务

Resource server：存放Resource Owner的资源的服务器，一般指数据库的服务器



参考

https://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html

https://www.ruanyifeng.com/blog/2019/04/oauth_design.html

https://www.ruanyifeng.com/blog/2019/04/oauth-grant-types.html

