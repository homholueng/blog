---
title: Chapter 3 - 基本的 bash shell 命令
date: 2019-04-20 15:05:03
tags:
  - Linux
  - Shell
categories:
  - Linux CL and Shell Programing
---

## 3.4 浏览文件系统

### 3.4.1 Linux 文件系统

Linux 虚拟目录中比较复杂的部分是它如何协调管理各个存储设备。在 Linux PC 上安装的第 一块硬盘称为根驱动器。根驱动器包含了虚拟目录的核心，其他目录都是从那里开始构建的。

Linux 会在根驱动器上创建一些特别的目录，我们称之为挂载点（mount point）。挂载点是虚拟目录中用于分配额外存储设备的目录。虚拟目录会让文件和目录出现在这些挂载点目录中，然 而实际上它们却存储在另外一个驱动器中。

## 3.6 处理文件

### 3.6.1 创建文件

使用 `touch` 命令不仅能够创建，文件，还能够改变文件的修改时间：

```shell
$ ls -l test_one 
-rw-rw-r-- 1 christine christine 0 May 21 14:17 test_one 
$ touch test_one 
$ ls -l test_one 
-rw-rw-r-- 1 christine christine 0 May 21 14:35 test_one 
$
```

要注意的是，如果只使用 `ls –l` 命令，并不会显示访问时间。这是因为默 认显示的是修改时间。要想查看文件的访问时间，需要加入另外一个参数：`--time=atime`。有 了这个参数，就能够显示出已经更改过的文件访问时间。

## 3.8 查看文件内容

### 3.8.1 查看文件类型

`file` 命令是一个随手可得的便捷工具, 不仅能确定文件中包含的文本信息，还能确定该文本文件的字符编码。

### 3.8.2 查看整个文件

`cat` 命令加上 `-n` 参数会在展示文件内容的同时加上行号。

如果想只给有文本的地方加上行号，可以使用 `-b` 选项。


### 3.8.3 查看部分文件

使用 `tail` 命令会显示文件最后几行的内容，默认情况下，它会显示文件的末尾10行。

`-f` 参数是 tail 命令的一个突出特性。它允许你在其他进程使用该文件时查看文件的内容。`tail` 命令会保持活动状态，并不断显示添加到文件中的内容。

而 `head` 命令会现实文件开头的内容，默认情况下，它会显示文件前10行的文本。

