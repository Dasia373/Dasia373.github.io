+++
date = '2026-03-10T21:31:33+08:00'
draft = false
title = 'Linux基本操作指令'

tags: ["Linux"]

+++

# Linux基本操作：

### 1. 目录与文件操作命令

- **pwd**:

  **示例场景**: 执行 `pwd` 可以显示当前所在路径，例如输出 `/home/book` 。

- **cd**:

  **示例场景**: 执行 `cd /home/` 可以切换到 `/home` 路径 。

- **mkdir**:

  **示例场景 1**: 执行 `mkdir dir0` 创建一个目录 。

  **示例场景 2**: 执行 `mkdir -p dir1/dir2` 创建目录及子目录 。

- **rmdir**:

  **示例场景**: 执行 `rmdir dir1` 删除一个空目录 。注意，不能删除一个非空目录 。

- **ls**:

  **示例场景 1**: 执行 `ls` 显示当前目录下文件 。

  **示例场景 2**: 执行 `ls -l` 显示文件更完整信息 。

  **示例场景 3**: 执行 `ls -a` 显示当前目录下文件及隐藏文件 。

  **示例场景 4**: 执行 `ls -lh`，文件大小会以 K/M/G 等可读方式列出来 。

- **cp**:

  **示例场景**: 执行 `cp -r dir1/ dir2` 复制 dir1 文件夹 。

- **rm**:

  **示例场景 1**: 执行 `rm file1` 删除文件 。

  **示例场景 2**: 执行 `rm -r dir1/` 删除文件夹 。

- **cat**:

  **示例场景**: 执行 `cat file1.txt file2.txt` 串联文件并依次全部打印在标准输出中 。

- **touch**:

  **示例场景**: 执行 `touch file` 创建一个名为 file 的文件 。

------

### 2. 文件权限与属性修改命令

- **chgrp**:

  **示例场景**: 执行 `chgrp hy install.log` 将 install.log 文件的用户组改为 hy 用户组 。

- **chown**:

  **示例场景 1**: 执行 `chown bin install.log` 改变文件所有者 。

  **示例场景 2**: 执行 `chown book:book install.log` 改变文件的所有者和用户组 。

- **chmod**:

  **示例场景 1**: 执行 `chmod 777 .bashrc` 将 .bashrc 文件的所有权限设置都启用 。

  **示例场景 2**: 执行 `chmod u=rwx,go=rx .bashrc` 通过符号类型改变文件权限 。

  **示例场景 3**: 执行 `chmod a+w .bashrc` 表示为所有身份添加写权限 。

------

### 3. 查找与搜索命令

- **find**:

  **示例场景 1**: 执行 `find /home/book/dira/ -name "test1.txt "` 查找指定目录下的 test1.txt 文件 。

  **示例场景 2**: 执行 `find /home/book -mtime -2` 查找 /home 目录下两天内有变动的文件 。

- **grep**:

  **示例场景 1**: 执行 `grep -n "abc" test1.txt` 在 test1.txt 中查找字符串 abc 并显示目标位置的行号 。

  **示例场景 2**: 执行 `grep "ABC" * -nR | grep "\.h"` 可以在当前目录下搜索包含有 ABC 的头文件 。

------

### 4. 压缩与解压缩命令

- **gzip**:

  **示例场景 1**: 执行 `gzip -k mypwd.1` 得到一个 .gz 结尾的压缩文件并保留源文件 。

  **示例场景 2**: 执行 `gzip -l pwd.1.gz` 查看压缩文件 。

  **示例场景 3**: 执行 `gzip -kd pwd.1.gz` 解压文件 。

- **bzip2**:

  **示例场景 1**: 执行 `bzip2 -k mypwd.1` 得到一个 .bz2 后缀的压缩文件并保留源文件 。

  **示例场景 2**: 执行 `bzip2 -kd mypwd.1.bz2` 解压文件 。

- **tar**:

  **示例场景 1**: 执行 `tar czvf dira.tar.gz dira` 把目录 dira 压缩、打包为 dira.tar.gz 文件 。

  **示例场景 2**: 执行 `tar xzvf dira.tar.gz -C /home/book` 将文件解压到 /home/book 目录 。

  **示例场景 3**: 执行 `tar cjvf dira.tar.bz2 dira` 把目录 dira 压缩、打包为 dira.tar.bz2 文件 。

------

### 5. 网络命令

- **ifconfig**:

  **示例场景 1**: 执行 `ifconfig` 查看当前正在使用的网卡 。

  **示例场景 2**: 执行 `sudo ifconfig ens160 192.168.1.137` 设置网卡 IP 。

- **ping**:

  **示例场景**: 执行 `ping 8.8.8.8` 或 `ping www.baidu.com` 测试网络连通性 。

- **route**:

  **示例场景 1**: 执行 `sudo route add default gw 192.168.1.1` 添加默认网关路由 。

  **示例场景 2**: 执行 `sudo route del default gw 192.168.1.1` 删除路由 。



