---
title: Chapter 4 - 更多的 bash shell 命令
date: 2019-04-20 15:06:12
tags:
  - Linux
  - Shell
categories:
  - Linux CL and Shell Programing
---

## 管理进程

### 4.1 检测程序

#### 4.1.1 探查进程

Linux 系统中使用的 GNU ps 命令支持3种不同类型的命令行参数：
- Unix 风格的参数，前面加单破折线
- BSD 风格的参数，前面不加破折线
- GNU 风格的长参数，前面加双破折线

##### Unix 风格的参数

使用 ps 命令的关键不在于记住所有可用的参数，而在于记住最有用的那些参数。

如果你想查看系统上运行的所有进程，可用 `-ef` 参数组合，`-e` 参数指定显示所有运行在系统上的进程；`-f` 参数则扩展了输出：

```shell
$ ps -ef 
  UID   PID  PPID   C STIME   TTY           TIME CMD
    0     1     0   0 29 119  ??         4:42.80 /sbin/launchd
    0    66     1   0 29 119  ??         0:07.86 /usr/sbin/syslogd
    0    67     1   0 29 119  ??         0:04.20 /usr/libexec/UserEventAgent (System)
    0    70     1   0 29 119  ??         0:03.37 /System/Library/PrivateFrameworks/Uninstall.framework/Resources/uninstalld
    0    71     1   0 29 119  ??         0:04.53 /usr/libexec/kextd
    0    72     1   0 29 119  ??         0:49.94 /System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/FSEvents.framework/Versions/A/Support/fseventsd
    ...
```

- UID：启动这些进程的用户
- PID：进程的进程 ID
- PPID：父进程的进程 ID
- C：进程声明周期中的 CPU 利用率
- STIME：进程启动时的系统时间
- TTY：进程启动时的终端设备
- TIME：运行进程需要的累积 CPU 时间
- CMD：启动进程的程序名

如果需要获取更多信息，可以使用 `-l` 参数：

```shell
$ ps -l
  UID   PID  PPID        F CPU PRI NI       SZ    RSS WCHAN     S             ADDR TTY           TIME CMD
  501 10829 10827     4006   0  31  0  4318368  14308 -      Ss                  0 ttys001    0:00.06 /Applications/iTerm (3.1.5.beta
  501 10831 10830     4006   0  31  0  4298784   1340 -      S                   0 ttys001    0:00.76 -zsh
  501 11226 10831     4006   0  31  0 558515572  21996 -      S+                  0 ttys001    0:01.33 docker run -i -t ubuntu /bin/ba
  501 11228 10827     4006   0  31  0  4318368  14308 -      Ss                  0 ttys002    0:00.07 /Applications/iTerm (3.1.5.beta
  501 11230 11229     4006   0  31  0  4297760   3804 -      S                   0 ttys002    0:00.62 -zsh
```

- F：内核分配给进程的系统标记
- S：进程的状态（O 代表正在运行；S 代表在休眠；R 代表可运行，正等待运行；Z 代表僵化，进程已结束但父进程已不存在；T 代表停止）
- PRI：进程的优先级（越大的数字代表越低的优先级）
- NI：谦让度值用来参与决定优先级
- ADDR：进程的内存地址
- SZ：假如进程被换出，所需交换空间的大致大小
- WCHAN：进程休眠的内核函数的地址

### 4.1.2 实时监测进程

ps 命令虽然在收集运行在系统上的进程信息时非常有用，但也有不足之处：它只能显示 某个特定时间点的信息。如果想观察那些频繁换进换出的内存的进程趋势，用 ps 命令就不方 便了。

而 top 命令刚好适用这种情况。top 命令跟 ps 命令相似，能够显示进程信息，但它是实时显示的。

其显示内容如下

```text
top - 07:15:02 up 1 day,  8:20,  0 users,  load average: 0.28, 0.38, 0.40
Tasks:   2 total,   1 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.5 us,  2.5 sy,  0.0 ni, 96.9 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  2047036 total,   188520 free,   968420 used,   890096 buff/cache
KiB Swap:  1048572 total,  1037316 free,    11256 used.   913360 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    1 root      20   0   18504   3376   2964 S   0.0  0.2   0:00.05 bash
   11 root      20   0   36616   3076   2592 R   0.0  0.2   0:00.01 top
```

##### part 1

输出的第一部分显示的是系统的概况：第一行显示了当前时间、系统的运行时间、登录的用户数以及系统的平均负载

平均负载有3个值：最近1分钟的、最近5分钟的和最近15分钟的平均负载。值越大说明系统的负载越高。

##### part 2 

第二行显示了进程概要信息——top 命令的输出中将进程叫作任务（task）：有多少进程处在运行、休眠、停止或是僵化状态（僵化状态是指进程完成了，但父进程没有响应）。

##### part 3

下一行显示了 CPU 的概要信息。top 根据进程的属主（用户还是系统）和进程的状态（运行、空闲还是等待）将 CPU 利用率分成几类输出。

##### part 4

紧跟其后的两行说明了系统内存的状态。第一行说的是系统的物理内存：总共有多少内存，当前用了多少，还有多少空闲。后一行说的是同样的信息，不过是针对系统交换空间（如果分配了的话）的状态而言的。

##### part 5

最后一部分显示了当前运行中的进程的详细列表，有些列跟 ps 命令的输出类似。

- PID：进程的 ID
- USER：进程属主的名字
- PR：进程的优先级
- NI：进程的谦让度值
- VIRT：进程占用的虚拟内存总量
- RES：进程占用的物理内存总量
- SHR：进程和其他进程共享的内存总量
- S：进程的状态（D代表可中断的休眠状态，R代表在运行状态，S代表休眠状态，T代表跟踪状态或停止状态，Z代表僵化状态）
- %CPU：进程使用的CPU时间比例
- %MEM：进程使用的内存占可用内存的比例
- TIME+：自进程启动到目前为止的 CPU 时间总量
- COMMAND：进程所对应的命令行名称，也就是启动的程序名

默认情况下，top 命令在启动时会按照 %CPU 值对进程排序。键入 `f` 允许你选择对输出进行排序的字段，键入 `d` 允许你修改轮询间隔。

#### 4.1.3 结束进程

在 Linux 中，进程之间通过信号来通信。进程的信号就是预定义好的一个消息，进程能识别它并决定忽略还是作出反应。进程如何处理信号是由开发人员通过编程来决定的。

在 Linux 中有两个命令可以向运行中的进程发出进程信号。

##### 1. kill 命令

kill 命令可通过进程 ID（PID）给进程发信号。默认情况下，kill 命令会向命令行中列出的全部 PID 发送一个 TERM 信号。而且要发送进程信号，你必须是进程的属主或登录为 root 用户。

使用 `-s` 参数能够指定其他信号（用信号名或信号值）。


```shell
# kill -s HUP 3940
```

##### 2. killall 命令

killall 命令非常强大，它支持通过进程名而不是 PID 来结束进程。killall 命令也支持通配符，这在系统因负载过大而变得很慢时很有用。

```shell
# killall http*
```

上例中的命令结束了所有以 http 开头的进程，比如 Apache Web 服务器的 httpd 服务。

### 4.2 监测磁盘空间

#### 4.2.1 挂载存储媒体

如第3章中讨论的，Linux文件系统将所有的磁盘都并入一个虚拟目录下。在使用新的存储媒体之前，需要把它放到虚拟目录下。这项工作称为*挂载（mounting）*。

在今天的图形化桌面环境里，大多数 Linux 发行版都能自动挂载特定类型的可移动存储媒体。如果用的发行版不支持自动挂载和卸载可移动存储媒体，就必须手动完成。

##### 1. mount 命令

Linux 上用来挂载媒体的命令叫作 mount。默认情况下，mount 命令会输出当前系统上挂载的设备列表。

```shell
$ mount                                                                                                                   (env: usr)
/dev/disk1s1 on / (apfs, local, journaled)
devfs on /dev (devfs, local, nobrowse)
/dev/disk1s4 on /private/var/vm (apfs, local, noexec, journaled, noatime, nobrowse)
map -hosts on /net (autofs, nosuid, automounted, nobrowse)
map auto_home on /home (autofs, automounted, nobrowse)
map -fstab on /Network/Servers (autofs, automounted, nobrowse)
```

mount 命令提供如下四部分信息：

- 媒体的设备文件名
- 媒体挂载到虚拟目录的挂载点
- 文件系统类型
- 已挂载媒体的访问状态

要手动在虚拟目录中挂载设备，需要以 root 用户身份登录，或是以 root 用户身份运行 mount 命令。下面是手动挂载媒体设备的基本命令：

```
mount -t type device directory
```

`type` 参数指定了磁盘被格式化的文件系统类型。

##### 2. umount 命令

从 Linux 系统上移除一个可移动设备时，不能直接从系统上移除，而应该先卸载。

卸载设备的命令是 umount，其格式非常简单：

```
umount [directory | device ]
```

umount 命令支持通过设备文件或者是挂载点来指定要卸载的设备。

> 如果在卸载设备时，系统提示设备繁忙，无法卸载设备，通常是有进程还在访问该设备或使用该设备上的文件。这时可用 lsof 命令获得使用它的进程信息，然后在应用中停止使用该设备或停止该进程。lsof命令的用法很简单：`lsof /path/to/device/node`，或者 `lsof /path/to/mount/point`。

#### 4.2.2 使用 df 命令

有时你需要知道在某个设备上还有多少磁盘空间。df 命令可以让你很方便地查看所有已挂载磁盘的使用情况。

```shell
root@bd6467bc76d1:/# df
Filesystem     1K-blocks     Used Available Use% Mounted on
overlay         65792556 21643556  40777224  35% /
tmpfs              65536        0     65536   0% /dev
tmpfs            1023516        0   1023516   0% /sys/fs/cgroup
/dev/sda1       65792556 21643556  40777224  35% /etc/hosts
shm                65536        0     65536   0% /dev/shm
tmpfs            1023516        0   1023516   0% /proc/acpi
tmpfs            1023516        0   1023516   0% /sys/firmware
```

每一列的含义如下：

- 设备的设备文件位置
- 能容纳多少个1024字节大小的块
- 已用了多少个1024字节大小的块
- 还有多少个1024字节大小的块可用
- 已用空间所占的比例
- 设备挂载到了哪个挂载点上

一个常用的参数是 `-h`。它会把输出中 的磁盘空间按照用户易读的形式显示。

#### 使用 du 命令

du 命令可以显示某个特定目录（默认情况下是当前目录）的磁盘使用情况。这一方法可用来快速判断系统上某个目录下是不是有超大文件。

```shell
root@bd6467bc76d1:/usr# du
11272	./bin
504	./share/info
4	./share/terminfo
8	./share/bug/init-system-helpers
8	./share/bug/apt
8	./share/bug/procps
28	./share/bug
16	./share/locale/pa/LC_MESSAGES
20	./share/locale/pa
256	./share/locale/ja/LC_MESSAGES
260	./share/locale/ja
188	./share/locale/sk/LC_MESSAGES
...
```

每行输出左边的数值是每个文件或目录占用的磁盘块数。注意，这个列表是从目录层级的最底部开始，然后按文件、子目录、目录逐级向上。

下面是能让du命令用起来更方便的几个命令行参数：

- `-c`：显示所有已列出文件总的大小
- `-h`：按用户易读的格式输出大小
- `-d n`：指定显示的目录层级深度


### 4.3 处理数据文件

#### 4.3.1 排序数据

处理大量数据时的一个常用命令是 sort 命令。默认情况下，sort 命令按照会话指定的默认语言的排序规则对文本文件中的数据行排序。

```shell
$ cat file1
one
two
three
four
five
$ sort file1
five
four
one
three
two
```

默认情况下，sort 命令会把数字当做字符来执行标准的字符排序，产生的输出可能根本就不是你要的。解决这个问题可用 `-n` 参数，它会告诉 sort 命令不要把数字识别成字符，并且按值排序。

`-k` 和 `-t` 参数在对按字段分隔的数据进行排序时非常有用，例如 `/etc/passwd` 文件。可以用 `-t` 参数来指定字段分隔符，然后用 `-k` 参数来指定排序的字段。

```shell
$ sort -t ':' -k 3 -n /etc/passwd
root:*:0:0:System Administrator:/var/root:/bin/sh
daemon:*:1:1:System Services:/var/root:/usr/bin/false
_uucp:*:4:4:Unix to Unix Copy Protocol:/var/spool/uucp:/usr/sbin/uucico
_taskgated:*:13:13:Task Gate Daemon:/var/empty:/usr/bin/false
_networkd:*:24:24:Network Services:/var/networkd:/usr/bin/false
_installassistant:*:25:25:Install Assistant:/var/empty:/usr/bin/false
...
```

下面是一个将 du 和 sort 命令组合起来使用的例子（sort 的 `-r` 参数将结果按降序输出）：

```shell
root@bd6467bc76d1:/usr# du -d 1 | sort -nr
50612	.
26148	./lib
11272	./bin
11260	./share
1872	./sbin
44	./local
4	./src
4	./include
4	./games
```

#### 4.3.2 搜索数据

你会经常需要在大文件中找一行数据，而这行数据又埋藏在文件的中间。这时并不需要手动翻看整个文件，用 grep 命令来帮助查找就行了。grep 命令的命令行格式如下。默认情况下，grep 命令用基本的 Unix 风格正则表达式来匹配模式。

```
grep [options] pattern [file]
```

grep 命令会在输入或指定的文件中查找包含匹配指定模式的字符的行。grep 的输出就是包含了匹配模式的行。

如果要进行反向搜索（输出不匹配该模式的行），可加 `-v` 参数。

如果要显示匹配模式的行所在的行号，可加`-n` 参数。

如果只要知道有多少行含有匹配的模式，可用 `-c` 参数。

如果要指定多个匹配模式，可用 `-e` 参数来指定每个模式：

```shell
$ grep -e t -e f file1
two
three
four
five
```

egrep 命令是 grep 的一个衍生，支持 POSIX 扩展正则表达式。POSIX 扩展正则表达式含有更多的可以用来指定匹配模式的字符。

fgrep 则是另外一个版本，支持将匹配模式指定为用换行符分隔的一列固定长度的字符串。这样就可以把这列字符串放到一个文件中，然后在 fgrep 命令中用其在一个大型文件中搜索字符串了。

#### 4.3.3 压缩数据

gzip 是 Linux 上最流行的压缩工具。 其包含的工具如下：

- gzip：用来压缩文件
- gzcat：用来查看压缩过的文本文件内容
- gunzip：用来解压文件

#### 4.3.4 归档数据

目前，Unix 和 Linux 上最广泛使用的归档工具是 tar 命令。

```
tar function [options] object1 object2 ...
```

function 参数定义了 tar 命令应该做什么：

- `-A`：将一个已有 tar 归档文件追加到另一个已有 tar 归档文件中
- `-c`：创建一个新的 tar 归档文件
- `-r`：追加文件到已有 tar 归档文件末尾
- `-t`：列出已有 tar 归档文件的内容
- `-u`：将比 tar 归档文件中已有的同名文件新的文件追加到该 tar 归档文件中
- `-x`：从已有 tar 归档文件中提取文件

上述每个功能可用选项来针对 tar 归档文件定义一个特定行为：

- `-c`：切换到指定目录
- `-f`：输出结果到文件或设备文件
- `-j`：将输出重定向给 bzip2 命令来压缩内容
- `-p`：保留所有文件权限
- `-v`：在处理文件时显示文件
- `-z`：将输出重定向给 gzip 命令来压缩内容

> 下载了开源软件之后，你会经常看到文件名以 .tgz 结尾。这些是 gzip 压缩过的 tar 文件，可以用命令 `tar -zxvf filename.tgz` 来解压。