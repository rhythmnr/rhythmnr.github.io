---
title: 更改通过docker运行的kibana的显示语言
date: 2022-9-16 17:04:00
categories:
- 运维
---
因为kibana是通过 docker compose启动的，docker compose通过`I18N_LOCALE: "zh-CN"`定义了显示语言为中文：

```yaml
version: "3.5"
  
services:
  elasticsearch:
     container_name: elasticsearch
     image: elasticsearch:7.6.0           
     restart: always                        #重启方式
     environment:
       discovery.type: single-node          #环境变量：运行模式 单例
     ports:
       - "9200:9200"                        #端口映射
       - "9300:9300"
  kibana:
    container_name: kibana
    image: kibana:7.6.0                    
    restart: always                         #重启方式
    environment:
      I18N_LOCALE: "zh-CN"                  #指定中文
    ports:
       - "5601:5601"                      
```

所以在浏览器查看运行在5601端口的kibana服务，显示语言为中文：

![WX20220916-113526@2x](https://tvax1.sinaimg.cn/large/006gLprLgy1h68ajiev8ej325k0rmh25.jpg)

因为中文下有些乱码和显示错误，比如：

![WX20220916-113740@2x](https://tvax1.sinaimg.cn/large/006gLprLgy1h68alim2vmj32380qin4h.jpg)

所以想要改回显示成英文。

在网上搜寻了“docker kibana更改语言"，根据搜到的结果，试了两个方法：

**方法1**

停止kibana服务，在docker-compose文件中取消指定`I18N_LOCALE: "zh-CN"`，然后`docker-compose restart kibana`，发现不生效

**方法2**

:`docker exec -it kibana bash`

`cd cd config/`

`vi kibana.yml `

添加`i18n.locale: "zh-CN"`

然后`exit`退出容器

最后`docker restart kibana`，发现也还是不行。

两个方法都不行，那可能是更改对于docker没有生效，于是搜索“docker配置变更不生效“。

发现是需要重启docker，停止容器后在docker compose中修改配置，然后`systemctl restart docker`，重启后执行docker compose up即可。