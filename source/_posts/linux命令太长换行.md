---
title: linux命令太长换行
date: 2022-9-28 00:00:00
tags:
- 原创
categories:
- 运维
---

可以在末尾加上`  \`，然后写第二行，如下：

```shell
docker run -d --name=service-test --net=host \
-v $PWD/config/config.yml:/usr/local/app/config.yml \
-v $PWD/logs:/usr/local/app/logs \
service -c=config.yml
```

直接把这个命令粘贴到Terminal后，运行效果如下：

```shell
[root@n-test remote]# docker run -d --name=service-test --net=host \
> -v $PWD/config/config.yml:/usr/local/app/config.yml \
> -v $PWD/logs:/usr/local/app/logs \
> service -c=config.yml
a0067d0fa6d98b7d9391fdb1d4bf877f42ba86e53e38befebc6871d9866d2c37
```

