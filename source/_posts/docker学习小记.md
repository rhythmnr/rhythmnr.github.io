---
title: docker学习小记
date: 2022-9-1 16:14:00
categories:
- 运维
---
docker学习小记

## Dockerfile命令解析

### COPY

语法

```shell
COPY [--chown=<user>:<group>] <源路径>... <目标路径>
COPY [--chown=<user>:<group>] ["<源路径1>",... "<目标路径>"]
```

`<目标路径>` 可以是容器内的绝对路径，也可以是相对于工作目录的相对路径（工作目录可以用 `WORKDIR` 指令来指定）。目标路径不需要事先创建，如果目录不存在会在复制文件前先行创建缺失目录。

使用 `COPY` 指令，源文件的各种元数据都会保留。比如读、写、执行权限、文件变更时间等。

如果源路径为文件夹，复制的时候不是直接复制该文件夹，而是将文件夹中的内容复制到目标路径。

###ADD

在COPY基础上多加了一些命令

这个功能其实并不实用，而且不推荐使用。

比如 `<源路径>` 可以是一个 `URL`，这种情况下，Docker 引擎会试图去下载这个链接的文件放到 `<目标路径>` 去。此外如果 `<源路径>` 为一个 `tar` 压缩文件的话，压缩格式为 `gzip`, `bzip2` 以及 `xz` 的情况下，`ADD` 指令将会自动解压缩这个压缩文件到 `<目标路径>` 去

###CMD



### ENTRYPOINT

配置容器启动后执行的命令，并且不可被 docker run 提供的参数覆盖。



### 我的使用🌟

使得docker run支持go flag参数，-c=xxx参数也就是程序中的参数，和go run的参数一致。但是在docker run时，-c的路径是容器内的路径，所以如果容器内没有的话，需要挂载，容器内目录必须是绝对路径，宿主机目录不存在时会生成。通过-v挂载，`-v 宿主机路径:容器内路径`

```shell
docker run -v /home/ftpuser/flora-gopacket-service/config.yml:/usr/src/app/config.yml packet-go2 -c=/usr/src/app/config.yml
```

好像只有Dockerfile里运行命令的参数写ENTRYPOINT，-c才有效，上面的镜像对应的Dockerfile如下：

```Dockerfile
FROM golang:1.18
ENV GO111MODULE=on \
    GOPROXY=https://goproxy.cn,direct \
		PORT=80
RUN apt-get update && echo "y" | apt-get upgrade
RUN echo "y" | apt-get install libpcap-dev
WORKDIR /usr/src/app
COPY go.mod go.sum ./
RUN go mod download && go mod verify
COPY . .
RUN go build -v -o /usr/local/bin/ ./...
ENTRYPOINT ["/usr/local/bin/flora-gopacket-service"] # 注意这一行
```

基于alpine，可以构建出可以使用的gopacket服务，对应的镜像是：

```shell
 docker images|grep gopacket                                                                                                                2 ↵  10074  15:50:45
gopacket                   alpine-ok   6b9e79fe7834   About a minute ago   31.8MB
```

这个镜像是我在基于如下Dockerfile的容器的基础上改的：

```shell
FROM alpine:3.4
RUN mkdir /lib64 && ln -s /lib/libc.musl-x86_64.so.1 /lib64/ld-linux-x86-64.so.2
WORKDIR /usr/local/bin
```

改的操作如下：

> 参考了下[ libpcap.so.0.8相关报错](https://github.com/knownsec/ksubdomain/issues/1)
>
> 解答：
>
> 我是这样解决的：
> 先直接yum安装libpcap-devel：
> yum install libpcap-devel
> 然后locate一下，发现了安装的是1.5.3版本，定位出/usr/lib64目录下的三个文件：
> locate libpcap
> /usr/lib64/libpcap.so
> /usr/lib64/libpcap.so.1
> /usr/lib64/libpcap.so.1.5.3
> 然后cd到/usr/lib64目录下，ls看一下那三个文件，发现libpcap.so和libpcap.so.1都是libpcap.so.1.5.3的软链接文件。
> 既然这样，那就再建一个软链接文件就好了：
> ln -s libpcap.so.1.5.3 libpcap.so.0.8

````shell
docker run -it zz    127 ↵  10068  15:20:16
/usr/local/bin # apk --help
apk-tools 2.12.7, compiled for x86_64.

usage: apk [<OPTIONS>...] COMMAND [<ARGUMENTS>...]

Package installation and removal:
  add        Add packages to WORLD and commit changes
  del        Remove packages from WORLD and commit changes

System maintenance:
  fix        Fix, reinstall or upgrade packages without modifying WORLD
  update     Update repository indexes
  upgrade    Install upgrades available from repositories
  cache      Manage the local package cache

Querying package information:
  info       Give detailed information about packages or repositories
  list       List packages matching a pattern or other criteria
  dot        Render dependencies as graphviz graphs
  policy     Show repository policy for packages
  search     Search for packages by name or description

Repository maintenance:
  index      Create repository index file from packages
  fetch      Download packages from global repositories to a local directory
  manifest   Show checksums of package contents
  verify     Verify package integrity and signature

Miscellaneous:
  audit      Audit system for changes
  stats      Show statistics about repositories and installations
  version    Compare package versions or perform tests on version strings

This apk has coffee making abilities.
For more information: man 8 apk
/usr/local/bin # apk add ^C
/usr/local/bin # apk^C
/usr/local/bin # locate libpcap
/bin/sh: locate: not found
/usr/local/bin # apk add libpcap libpcap-dev
fetch https://dl-cdn.alpinelinux.org/alpine/v3.14/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.14/community/x86_64/APKINDEX.tar.gz
(1/3) Installing libpcap (1.10.0-r0)
(2/3) Installing pkgconf (1.7.4-r0)
(3/3) Installing libpcap-dev (1.10.0-r0)
Executing busybox-1.33.1-r8.trigger
OK: 7 MiB in 17 packages
/usr/local/bin # locate libpcap
/bin/sh: locate: not found
/usr/local/bin # cd /usr/lib64/
/bin/sh: cd: can't cd to /usr/lib64/: No such file or directory
/usr/local/bin # ls -S /usr/lib64/libpcap.so /usr/lib64/libpcap.so.0.8
/usr/local/bin # apk info -a  libpcap
libpcap-1.10.0-r0 description:
A system-independent interface for user-level packet capture

libpcap-1.10.0-r0 webpage:
https://www.tcpdump.org/

libpcap-1.10.0-r0 installed size:
256 KiB

libpcap-1.10.0-r0 depends on:
so:libc.musl-x86_64.so.1

libpcap-1.10.0-r0 provides:
so:libpcap.so.1=1.10.0

libpcap-1.10.0-r0 is required by:
libpcap-dev-1.10.0-r0

libpcap-1.10.0-r0 contains:
usr/lib/libpcap.so.1
usr/lib/libpcap.so.1.10.0

libpcap-1.10.0-r0 triggers:

libpcap-1.10.0-r0 has auto-install rule:

libpcap-1.10.0-r0 affects auto-installation of:

libpcap-1.10.0-r0 replaces:

libpcap-1.10.0-r0 license:
BSD-3-Clause

/usr/local/bin # ls
unix-flora-gopacket-service
/usr/local/bin #  apk info
musl
busybox
alpine-baselayout
alpine-keys
libcrypto1.1
libssl1.1
ca-certificates-bundle
libretls
ssl_client
zlib
apk-tools
scanelf
musl-utils
libc-utils
libpcap
pkgconf
libpcap-dev
/usr/local/bin # ls
unix-flora-gopacket-service
/usr/local/bin # ./unix-flora-gopacket-service 
Error loading shared library libpcap.so.0.8: No such file or directory (needed by ./unix-flora-gopacket-service)
Error relocating ./unix-flora-gopacket-service: pcap_list_tstamp_types: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_set_promisc: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_tstamp_type_name_to_val: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_findalldevs: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_sendpacket: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_close: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_set_tstamp_precision: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_list_datalinks: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_open_live: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_setdirection: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_geterr: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_fopen_offline_with_tstamp_precision: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_statustostr: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_set_buffer_size: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_compile: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_get_selectable_fd: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_next_ex: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_set_timeout: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_set_immediate_mode: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_freealldevs: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_stats: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_set_snaplen: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_lookupnet: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_set_datalink: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_free_datalinks: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_create: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_tstamp_type_val_to_name: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_datalink: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_offline_filter: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_activate: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_setfilter: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_set_tstamp_type: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_datalink_val_to_name: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_free_tstamp_types: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_open_dead: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_setnonblock: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_freecode: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_can_set_rfmon: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_datalink_name_to_val: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_snapshot: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_datalink_val_to_description: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_get_tstamp_precision: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_lib_version: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_set_rfmon: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_open_offline_with_tstamp_precision: symbol not found
/usr/local/bin # locate
/bin/sh: locate: not found
/usr/local/bin # apk add locate
ERROR: unable to select packages:
  locate (no such package):
    required by: world[locate]
/usr/local/bin # cd /usr
/usr # ls
bin      include  lib      local    sbin     share
/usr # cd lib
/usr/lib # ls
engines-1.1          libpcap.so           libpkgconf.so.3      libtls.so.2          pkgconfig
libcrypto.so.1.1     libpcap.so.1         libpkgconf.so.3.0.0  libtls.so.2.0.3
libpcap.a            libpcap.so.1.10.0    libssl.so.1.1        modules-load.d
/usr/lib # ls|grep libpcap
libpcap.a
libpcap.so
libpcap.so.1
libpcap.so.1.10.0
/usr/lib # ls -S /usr/lib64/libpcap.so /usr/lib64/libpcap.so.0.8
ls: /usr/lib64/libpcap.so: No such file or directory
ls: /usr/lib64/libpcap.so.0.8: No such file or directory
/usr/lib # ln  -S /usr/lib64/libpcap.so /usr/lib64/libpcap.so.0.8
ln: /usr/lib64/libpcap.so.0.8: No such file or directory
/usr/lib # l -S /usr/lib64/libpcap.so /usr/lib64/libpcap.so.0.8
/bin/sh: l: not found
/usr/lib # ln -S /usr/lib64/libpcap.so /usr/lib64/libpcap.so.0.8
ln: /usr/lib64/libpcap.so.0.8: No such file or directory
/usr/lib # ls
engines-1.1          libpcap.so           libpkgconf.so.3      libtls.so.2          pkgconfig
libcrypto.so.1.1     libpcap.so.1         libpkgconf.so.3.0.0  libtls.so.2.0.3
libpcap.a            libpcap.so.1.10.0    libssl.so.1.1        modules-load.d
/usr/lib # touch libpcap.so.0.8
/usr/lib # ln -S /usr/lib64/libpcap.so /usr/lib64/libpcap.so.0.8
ln: /usr/lib64/libpcap.so.0.8: No such file or directory
/usr/lib # ls
engines-1.1          libpcap.so           libpcap.so.1.10.0    libssl.so.1.1        modules-load.d
libcrypto.so.1.1     libpcap.so.0.8       libpkgconf.so.3      libtls.so.2          pkgconfig
libpcap.a            libpcap.so.1         libpkgconf.so.3.0.0  libtls.so.2.0.3
/usr/lib # rm libpcap.so.0.8
/usr/lib # ls
engines-1.1          libpcap.so           libpkgconf.so.3      libtls.so.2          pkgconfig
libcrypto.so.1.1     libpcap.so.1         libpkgconf.so.3.0.0  libtls.so.2.0.3
libpcap.a            libpcap.so.1.10.0    libssl.so.1.1        modules-load.d
/usr/lib # ls
engines-1.1          libpcap.so           libpkgconf.so.3      libtls.so.2          pkgconfig
libcrypto.so.1.1     libpcap.so.1         libpkgconf.so.3.0.0  libtls.so.2.0.3
libpcap.a            libpcap.so.1.10.0    libssl.so.1.1        modules-load.d
/usr/lib # cd ..
/usr # ls
bin      include  lib      local    sbin     share
/usr # cd lib/
/usr/lib # ls -S /usr/lib/libpcap.so /usr/lib/libpcap.so.0.8
ls: /usr/lib/libpcap.so.0.8: No such file or directory
/usr/lib/libpcap.so
/usr/lib # ln -S /usr/lib/libpcap.so /usr/lib/libpcap.so.0.8
ln: /usr/lib/libpcap.so.0.8: No such file or directory
/usr/lib # touch libpcap.so.0.8
/usr/lib # ln -S /usr/lib/libpcap.so /usr/lib/libpcap.so.0.8
ln: libpcap.so.0.8: File exists
/usr/lib # touch libpcap.so.0.8^C
/usr/lib # cd -
/usr
/usr # ls
bin      include  lib      local    sbin     share
/usr # cd /usr/local/bin
/usr/local/bin # ls
unix-flora-gopacket-service
/usr/local/bin # ./unix-flora-gopacket-service 
Error loading shared library libpcap.so.0.8: Exec format error (needed by ./unix-flora-gopacket-service)
Error relocating ./unix-flora-gopacket-service: pcap_list_tstamp_types: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_set_promisc: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_tstamp_type_name_to_val: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_findalldevs: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_sendpacket: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_close: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_set_tstamp_precision: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_list_datalinks: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_open_live: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_setdirection: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_geterr: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_fopen_offline_with_tstamp_precision: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_statustostr: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_set_buffer_size: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_compile: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_get_selectable_fd: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_next_ex: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_set_timeout: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_set_immediate_mode: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_freealldevs: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_stats: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_set_snaplen: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_lookupnet: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_set_datalink: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_free_datalinks: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_create: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_tstamp_type_val_to_name: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_datalink: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_offline_filter: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_activate: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_setfilter: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_set_tstamp_type: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_datalink_val_to_name: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_free_tstamp_types: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_open_dead: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_setnonblock: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_freecode: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_can_set_rfmon: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_datalink_name_to_val: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_snapshot: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_datalink_val_to_description: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_get_tstamp_precision: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_lib_version: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_set_rfmon: symbol not found
Error relocating ./unix-flora-gopacket-service: pcap_open_offline_with_tstamp_precision: symbol not found
/usr/local/bin # cd /usr/lib/
/usr/lib # ls
engines-1.1          libpcap.so           libpcap.so.1.10.0    libssl.so.1.1        modules-load.d
libcrypto.so.1.1     libpcap.so.0.8       libpkgconf.so.3      libtls.so.2          pkgconfig
libpcap.a            libpcap.so.1         libpkgconf.so.3.0.0  libtls.so.2.0.3
/usr/lib # ls|grep lib
libcrypto.so.1.1
libpcap.a
libpcap.so
libpcap.so.0.8
libpcap.so.1
libpcap.so.1.10.0
libpkgconf.so.3
libpkgconf.so.3.0.0
libssl.so.1.1
libtls.so.2
libtls.so.2.0.3
/usr/lib # ls -l
total 796
drwxr-xr-x    2 root     root          4096 Jul 19 21:06 engines-1.1
lrwxrwxrwx    1 root     root            26 Jul 19 21:06 libcrypto.so.1.1 -> ../../lib/libcrypto.so.1.1
-rw-r--r--    1 root     root        413414 Jan  4  2021 libpcap.a
lrwxrwxrwx    1 root     root            12 Aug  5 07:29 libpcap.so -> libpcap.so.1
-rw-r--r--    1 root     root             0 Aug  5 07:42 libpcap.so.0.8
lrwxrwxrwx    1 root     root            17 Aug  5 07:29 libpcap.so.1 -> libpcap.so.1.10.0
-rwxr-xr-x    1 root     root        247824 Jan  4  2021 libpcap.so.1.10.0
lrwxrwxrwx    1 root     root            19 Aug  5 07:29 libpkgconf.so.3 -> libpkgconf.so.3.0.0
-rwxr-xr-x    1 root     root         64544 Mar 18  2021 libpkgconf.so.3.0.0
lrwxrwxrwx    1 root     root            23 Jul 19 21:06 libssl.so.1.1 -> ../../lib/libssl.so.1.1
lrwxrwxrwx    1 root     root            15 Jul 19 21:06 libtls.so.2 -> libtls.so.2.0.3
-rwxr-xr-x    1 root     root         71416 Mar 24 15:38 libtls.so.2.0.3
drwxr-xr-x    2 root     root          4096 Jul 19 21:06 modules-load.d
drwxr-xr-x    2 root     root          4096 Aug  5 07:29 pkgconfig
/usr/lib # ln -s libpcap.so.1.10.0 libpcap.so.0.8
ln: libpcap.so.0.8: File exists
/usr/lib # rm libpcap.so.0.8
/usr/lib # ln -s libpcap.so.1.10.0 libpcap.so.0.8
/usr/lib # cd /usr/local/bin
/usr/local/bin # ./unix-flora-gopacket-service 
panic: load config file error: open config.yml: no such file or directory

goroutine 1 [running]:
main.main()
        /usr/src/app/main.go:23 +0xf6c
/usr/local/bin # 
````

经过总结，最终完整可用的Dockerfile是

```Dockerfile
FROM alpine:3.4

RUN apk update
RUN mkdir /lib64 && ln -s /lib/libc.musl-x86_64.so.1 /lib64/ld-linux-x86-64.so.2
RUN apk add libpcap=1.7.4-r0 libpcap-dev=1.7.4-r0
WORKDIR /usr/lib
RUN ln -s libpcap.so.1.7.4 libpcap.so.0.8

WORKDIR /usr/local/bin
COPY flora-gopacket-service .

WORKDIR /usr/local/app

ENTRYPOINT ["flora-gopacket-service"]
```

那么这个flora-gopacket-service从哪里来呢？可以把项目复制到虚拟机上，然后执行go build，build出来了flora-gopacket-service，然后docker build -t xxx .即完成。

或者可以在项目下用下面的Dockerfile构建出镜像，再从容器里把flora-gopacket-service复制出来

```shell
FROM golang:1.18

ENV GO111MODULE=on \
    GOPROXY=https://goproxy.cn,direct \
		PORT=80

RUN apt-get update && echo "y" | apt-get upgrade
RUN echo "y" | apt-get install libpcap-dev
WORKDIR /usr/src/app

# RUN apt-get install libpcap-dev

# pre-copy/cache go.mod for pre-downloading dependencies and only redownloading them in subsequent builds if they change
COPY go.mod go.sum ./
RUN go mod download && go mod verify

COPY . .
RUN go build -v -o /usr/local/bin/ ./...

CMD ["flora-gopacket-service"]
```

基于这个镜像的可用`docker run`命令，注意要使用$PWD，使用./xxx或者直接xxx都不行，还有使用了host网络模式：

```shell
docker run -d --net=host -v $PWD/config/config.yml:/usr/local/app/config.yml -v $PWD/logs:/usr/local/app/logs  -v $PWD/pcap_file:/usr/local/app/pcap_file last0 -c=/usr/local/app/config.yml
```

## 构建轻量级的应用

可以使用alpine镜像，将二进制复制到镜像中。

参考

[Docker-从入门到实践](https://yeasy.gitbook.io/docker_practice/)    未看完，TODO再看看

## 补充

**linux查看操作系统版本的命令**，可以在容器内执行

```shell
cat /proc/version
```

golang条件编译



## go build使用

```shell
GOOS=linux GOARCH=amd64 go build        
# github.com/google/gopacket/pcap
../../../../go/pkg/mod/github.com/google/gopacket@v1.1.19/pcap/pcap.go:30:22: undefined: pcapErrorNotActivated
../../../../go/pkg/mod/github.com/google/gopacket@v1.1.19/pcap/pcap.go:52:17: undefined: pcapTPtr
../../../../go/pkg/mod/github.com/google/gopacket@v1.1.19/pcap/pcap.go:64:10: undefined: pcapPkthdr
../../../../go/pkg/mod/github.com/google/gopacket@v1.1.19/pcap/pcap.go:103:6: undefined: pcapBpfProgram
../../../../go/pkg/mod/github.com/google/gopacket@v1.1.19/pcap/pcap.go:110:7: undefined: pcapPkthdr
../../../../go/pkg/mod/github.com/google/gopacket@v1.1.19/pcap/pcap.go:268:33: undefined: pcapErrorActivated
../../../../go/pkg/mod/github.com/google/gopacket@v1.1.19/pcap/pcap.go:269:33: undefined: pcapWarningPromisc
../../../../go/pkg/mod/github.com/google/gopacket@v1.1.19/pcap/pcap.go:270:33: undefined: pcapErrorNoSuchDevice
../../../../go/pkg/mod/github.com/google/gopacket@v1.1.19/pcap/pcap.go:271:33: undefined: pcapErrorDenied
../../../../go/pkg/mod/github.com/google/gopacket@v1.1.19/pcap/pcap.go:272:33: undefined: pcapErrorNotUp
../../../../go/pkg/mod/github.com/google/gopacket@v1.1.19/pcap/pcap.go:272:33: too many errors
```

在Mac上，gopacket交叉编译报错

查了资料，说需要开启CGO，然后不能指定GOARCH

```shell
 CGO_ENABLED=1 GOOS=linux go build
# runtime/cgo
linux_syscall.c:67:13: error: implicit declaration of function 'setresgid' is invalid in C99 [-Werror,-Wimplicit-function-declaration]
linux_syscall.c:67:13: note: did you mean 'setregid'?
/Library/Developer/CommandLineTools/SDKs/MacOSX12.3.sdk/usr/include/unistd.h:593:6: note: 'setregid' declared here
linux_syscall.c:73:13: error: implicit declaration of function 'setresuid' is invalid in C99 [-Werror,-Wimplicit-function-declaration]
linux_syscall.c:73:13: note: did you mean 'setreuid'?
/Library/Developer/CommandLineTools/SDKs/MacOSX12.3.sdk/usr/include/unistd.h:595:6: note: 'setreuid' declared here
```

## 我的小记🌟

启动docker`sudo systemctl start docker`

docker-compose重启

`docker-compose -f xxx.yml restart`

复制容器内的文件到主机：`docker cp filebeat:/usr/share/filebeat ./filebeat/`

docker run -net=host指定网络模式为host只在linux上生效，[原文](https://stackoverflow.com/questions/52555007/docker-mac-alternative-to-net-host)

> The host networking driver only works on Linux hosts, and is not supported on Docker for Mac, Docker for Windows, or Docker EE for Windows Server.

