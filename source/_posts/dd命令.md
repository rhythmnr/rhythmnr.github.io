---
title: dd命令简介
date: 2024-02-21 18:00:00
categories:
- 运维
---

dd命令的名字来源于“数据复制”（data duplication）的缩写，可以用于数据备份，磁盘克隆，数据转换等

以下是 `dd` 命令的一般格式：

```
cssCopy code
dd [选项]
```

主要选项：

- `if=输入文件`：指定输入文件（input file），即源文件。
- `of=输出文件`：指定输出文件（output file），即目标文件。
- `bs=块大小`：指定每次读取或写入的数据块大小（block size）。
- `count=块数`：指定要处理的数据块数量。
- `skip=块数`：在输入文件中跳过指定数量的数据块后再开始操作。
- `seek=块数`：在输出文件中跳过指定数量的数据块后再开始写入。
- `status=进度输出频率`：指定进度输出频率，用于显示操作进度信息。
- `conv=选项`：指定转换操作，如 `conv=ucase`（将数据转换为大写）、`conv=lcase`（将数据转换为小写）等。

示例用法：

1. **复制文件**：

   ```
   dd if=inputfile of=outputfile
   ```

2. **从设备复制数据**：

   ```
   dd if=/dev/sda of=image.img
   ```

3. **创建指定大小的空白文件**：

   ```
   dd if=/dev/zero of=emptyfile bs=1M count=100
   ```

4. **备份和恢复磁盘**：

   ```
   # 备份整个磁盘
   dd if=/dev/sda of=/path/to/backup.img
   
   # 恢复整个磁盘
   dd if=/path/to/backup.img of=/dev/sda
   ```

5. **磁盘克隆**：

   ```
   dd if=/dev/sda of=/dev/sdb
   ```

6. 写入空字节流到文件，一般用于测试：

   ```shell
   dd if=/dev/zero of=/root/test.img bs=500M
   ```

   这条命令会将 `/dev/zero` 设备中的数据写入到 `/root/test.img` 文件中。`/dev/zero` 是一个特殊的设备文件，它提供了一个无限连续的空字节流，即写入到这个设备的数据都是由零字节组成的。

   因此，这条命令会将**零字节流**写入到 `/root/test.img` 文件中，其大小由 `bs=500M` 指定，表示每次写入的数据块大小为 500MB。这样就创建了一个名为 `test.img` 的文件，**并将其中填充了大量的零字节。**这会触发磁盘的写入操作，不会出发读取操作。