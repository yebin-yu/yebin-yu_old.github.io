- [什么是 Init 系统？init 系统的历史和现状](#什么是-init-系统init-系统的历史和现状)
    - [总结](#总结)
    - [什么是 Init 系统？](#什么是-init-系统)
    - [Init 系统的历史和现状](#init-系统的历史和现状)
  - [Sysvinit 概况](#sysvinit-概况)
    - [运行级别](#运行级别)
    - [sysvinit 启动](#sysvinit-启动)
        - [rc.sysinit](#rcsysinit)
        - [/ect/rcX.d](#ectrcxd)
        - [rc.local](#rclocal)
    - [sysvinit 系统关闭](#sysvinit-系统关闭)
    - [Init 进程的配置文件](#init-进程的配置文件)
- [systemd](#systemd)
    - [从 System V Init 到 Systemd](#从-system-v-init-到-systemd)
    - [设计理念](#设计理念)
      - [尽可能启动更少的进程](#尽可能启动更少的进程)
      - [尽可能将进程并行启动](#尽可能将进程并行启动)
      - [跟踪和管理进程的生命周期](#跟踪和管理进程的生命周期)
      - [统一管理服务日志](#统一管理服务日志)
    - [Unit 和 Target](#unit-和-target)
    - [日志管理](#日志管理)
    - [单元文件存放路径](#单元文件存放路径)
    - [命令汇总](#命令汇总)
    - [电源管理命令](#电源管理命令)
- [参考](#参考)

Linux 内核加载启动后，用户空间的第一个进程就是初始化进程，这个程序的物理文件约定位于 `/sbin/init`，当然也可以通过传递内核参数来让内核启动指定的程序。这个进程的特点是进程号为1，代表第一个运行的用户空间进程。不同发行版采用了不同的启动程序，主要有以下几种主流选择：

- 以7.0版本之前的 CentOS 为代表的 System V Init。
- 以 Ubuntu 为代表的 Linux 发行版采用 Upstart。
- CentOS7.0 版本的 Systemd。



# 什么是 Init 系统？init 系统的历史和现状

### 总结

Sysvinit 的优点是概念简单。Service 开发人员只需要编写启动和停止脚本，概念非常清楚；将 service 添加/删除到某个 runlevel 时，只需要执行一些创建/删除软连接文件的基本操作；这些都不需要学习额外的知识或特殊的定义语法(UpStart 和 Systemd 都需要用户学习新的定义系统初始化行为的语言)。

其次，sysvinit 的另一个重要优点是确定的执行顺序：脚本严格按照启动数字的大小顺序执行，一个执行完毕再执行下一个，这非常有益于错误排查。UpStart 和 systemd 支持并发启动，导致没有人可以确定地了解具体的启动顺序，排错不易。

但是串行地执行脚本导致 sysvinit 运行效率较慢，在新的 IT 环境下，启动快慢成为一个重要问题。此外动态设备加载等 Linux 新特性也暴露出 sysvinit 设计的一些问题。针对这些问题，人们开始想办法改进 sysvinit，以便加快启动时间，并解决 sysvinit 自身的设计问题。



### 什么是 Init 系统？

Linux 操作系统的启动首先从 BIOS 开始，接下来进入 boot loader，由 bootloader 载入内核，进行内核初始化。内核初始化的最后一步就是启动 pid 为 1 的 init 进程。这个进程是系统的第一个进程。它负责产生其他所有用户进程。

init 以守护进程方式存在，是所有其他进程的祖先。init 进程非常独特，能够完成其他进程无法完成的任务。

Init 系统能够定义、管理和控制 init 进程的行为。它负责组织和运行许多独立的或相关的始化工作(因此被称为 init 系统)，从而让计算机系统进入某种用户预订的运行模式。

仅仅将内核运行起来是毫无实际用途的，必须由 init 系统将系统代入可操作状态。比如启动外壳 shell 后，便有了人机交互，这样就可以让计算机执行一些预订程序完成有实际意义的任务。或者启动 X 图形系统以便提供更佳的人机界面，更加高效的完成任务。这里，字符界面的 shell 或者 X 系统都是一种预设的运行模式。

### Init 系统的历史和现状

大多数 Linux 发行版的 init 系统是和 System V 相兼容的，被称为 sysvinit。这是人们最熟悉的 init 系统。一些发行版如 Slackware 采用的是 BSD 风格 Init 系统，这种风格使用较少，本文不再涉及。其他的发行版如 Gentoo 是自己定制的。Ubuntu 和 RHEL 采用 upstart 替代了传统的 sysvinit。而 Fedora 从版本 15 开始使用了一个被称为 systemd 的新 init 系统。

可以看到不同的发行版采用了不同的 init 实现，本系列文章就是打算讲述三个主要的 Init 系统：sysvinit，UpStart 和 systemd。了解它们各自的设计特点，并简要介绍它们的使用。

在 Linux 主要应用于服务器和 PC 机的时代，SysVinit 运行非常良好，概念简单清晰。它主要依赖于 Shell 脚本，这就决定了它的最大弱点：启动太慢。在很少重新启动的 Server 上，这个缺点并不重要。而当 Linux 被应用到移动终端设备的时候，启动慢就成了一个大问题。为了更快地启动，人们开始改进 sysvinit，先后出现了 upstart 和 systemd 这两个主要的新一代 init 系统。Upstart 已经开发了 8 年多，在不少系统中已经替换 sysvinit。Systemd 出现较晚，但发展更快，大有取代 upstart 的趋势。

## Sysvinit 概况

sysvinit 就是 system V 风格的 init 系统，顾名思义，它源于 System V 系列 UNIX。它提供了比 BSD 风格 init 系统更高的灵活性。是已经风行了几十年的 UNIX init 系统，一直被各类 Linux 发行版所采用。

### 运行级别

Sysvinit 用术语 runlevel 来定义”预订的运行模式”。Sysvinit 检查 ‘/etc/inittab’ 文件中是否含有 ‘initdefault’ 项。 这告诉 init 系统是否有一个默认运行模式。如果没有默认的运行模式，那么用户将进入系统控制台，手动决定进入何种运行模式。

sysvinit 中运行模式描述了系统各种预订的运行模式。通常会有 8 种运行模式，即运行模式 0 到 6 和 S 或者 s。

每种 Linux 发行版对运行模式的定义都不太一样。但 0，1，6 却得到了大家的一致赞同：

- 0 关机
- 1 单用户模式
- 6 重启



### sysvinit 启动

ysvinit 巧妙地用脚本，文件命名规则和软链接来实现不同的 runlevel。首先，sysvinit 需要读取/etc/inittab 文件。分析这个文件的内容，它获得以下一些配置信息：

```
- 系统需要进入的 runlevel
- 捕获组合键的定义
- 定义电源 fail/restore 脚本
- 启动 getty 和虚拟控制台
```

得到配置信息后，sysvinit 顺序地执行以下这些步骤，从而将系统初始化为预订的 runlevel X。

```
- /etc/rc.d/rc.sysinit
- /etc/rc.d/rc 和/etc/rc.d/rcX.d/ (X 代表运行级别 0-6)
- /etc/rc.d/rc.local
- X Display Manager（如果需要的话）
```

##### rc.sysinit

首先，运行 rc.sysinit 以便执行一些重要的系统初始化任务。在 RedHat 公司的 RHEL5 中(RHEL6 已经使用 upstart 了)，rc.sysinit 主要完成以下这些工作。

```
- 激活 udev 和 selinux
- 设置定义在/etc/sysctl.conf 中的内核参数
- 设置系统时钟
- 加载 keymaps
- 使能交换分区
- 设置主机名(hostname)
- 根分区检查和 remount
- 激活 RAID 和 LVM 设备
- 开启磁盘配额
- 检查并挂载所有文件系统
- 清除过期的 locks 和 PID 文件
```

##### /ect/rcX.d

完成了以上这些工作之后，sysvinit 开始运行/etc/rc.d/rc 脚本。根据不同的 runlevel，rc 脚本将打开对应该 runlevel 的 rcX.d 目录(X 就是 runlevel)，找到并运行存放在该目录下的所有启动脚本。每个 runlevel X 都有一个这样的目录，目录名为/etc/rcX.d。

在这些目录下存放着很多不同的脚本。文件名以 S 开头的脚本就是启动时应该运行的脚本，S 后面跟的数字定义了这些脚本的执行顺序。在/etc/rc.d/rcX.d 目录下的脚本其实都是一些软链接文件，真实的脚本文件存放在/etc/init.d 目录下。如下所示：

```shell
[root@www ~]# ll /etc/rc5.d/
lrwxrwxrwx 1 root root 16 Sep  4  2008 K02dhcdbd -> ../init.d/dhcdbd
....(中间省略)....
lrwxrwxrwx 1 root root 14 Sep  4  2008 K91capi -> ../init.d/capi
lrwxrwxrwx 1 root root 23 Sep  4  2008 S00microcode_ctl -> ../init.d/microcode_ctl
lrwxrwxrwx 1 root root 22 Sep  4  2008 S02lvm2-monitor -> ../init.d/lvm2-monitor
....(中间省略)....
lrwxrwxrwx 1 root root 17 Sep  4  2008 S10network -> ../init.d/network
....(中间省略)....
lrwxrwxrwx 1 root root 11 Sep  4  2008 S99local -> ../rc.local
lrwxrwxrwx 1 root root 16 Sep  4  2008 S99smartd -> ../init.d/smartd
....(底下省略)....
```

##### rc.local

当所有的初始化脚本执行完毕。Sysvinit 运行/etc/rc.d/rc.local 脚本。

rc.local 是 Linux 留给用户进行个性化设置的地方。您可以把自己私人想设置和启动的东西放到这里，一台 Linux Server 的用户一般不止一个，所以才有这样的考虑。



### sysvinit 系统关闭

Sysvinit 不仅需要负责初始化系统，还需要负责关闭系统。在系统关闭时，为了保证数据的一致性，需要小心地按顺序进行结束和清理工作。

比如应该先停止对文件系统有读写操作的服务，然后再 umount 文件系统。否则数据就会丢失。

这种顺序的控制这也是依靠/etc/rc.d/rcX.d/目录下所有脚本的命名规则来控制的，在该目录下所有以 K 开头的脚本都将在关闭系统时调用，字母 K 之后的数字定义了它们的执行顺序。

这些脚本负责安全地停止服务或者其他的关闭工作。



### Init 进程的配置文件

| 参数                                          | 说明                                        |
| --------------------------------------------- | ------------------------------------------- |
| /etc/init.d/                                  | 服务启动脚本配置文件存放目录                |
| /etc/inittab                                  | 默认运行级别配置文件                        |
| /etc/init/rcS.conf                            | 系统初始化配置文件                          |
| /etc/init/rc.conf                             | 各运行级别初始化的配置文件                  |
| /etc/init/rcS-sulogin.conf                    | 单用户模式启动 /sbin/sushell 环境的配置文件 |
| /etc/init/control-alt-delete.conf             | 终端下的 ctrl+alt+del 热键操作的配置文件    |
| /etc/sysconfig/init                           | tty终端的配置文件                           |
| /etc/init/start-ttys.conf                     | 配置tty终端的开启数量、设备文件             |
| /etc/init/tty.conf  或  /etc/init/serial.conf | 控制tty终端的开启                           |





# systemd

### 从 System V Init 到 Systemd

System V Init 守护进程是一个基于运行级别的系统，它使用运行级别（单用户、多用户以及其他更多级别）和链接（位于 `/etc/rc?.d` 目录中，分别链接到 `/etc/init.d` 中的 Init 脚本）来启动和关闭系统服务。Upstart Init 守护进程则是基于事件的系统，它使用事件来启动和关闭系统服务。

近年来，Linux 系统的 Init 进程经历了两次重大的演进，传统的 System V Init 已经淡出历史舞台，新的 Init 系统 UpStart 和 Systemd 各有特点，而越来越多的 Linux 发行版采纳了 Systemd。

### 设计理念

#### 尽可能启动更少的进程

当 SysV-init 程序初始化系统时，必须将所有可能用到的后台服务进程全部运行起来。而用户需要等待系统将所有的服务都启动完成之后，才能够登陆。这种做法会带来两个问题：系统的启动时间过长和系统的资源浪费。

Systemd 提供了服务按需启动的能力，使得特定的服务只有在被真正请求时才会启动。特别是与具体硬件相关的服务，比如蓝牙服务仅在蓝牙适配器被插入时才需要运行，打印服务仅在打印机链接或程序要打印时才需要运行，甚至 sshd 服务也只需要用户在使用 SSH 连接到服务器时才需要启动。这种能力建立在 Systemd 对 DBus 总线或特定 Socket 端口监听的特性上的，这种设计相比于传统启动程序具有颠覆性进步。

#### 尽可能将进程并行启动

在 SysV-init 时代，将每个服务项目编号依次执行启动脚本。后来 Ubuntu 的 UPstart 解决了没有直接依赖启动项之间的并行启动。而 Systemd 通过 Socket 缓存、DBus 缓存和建立临时挂载点等方法进一步解决了启动进程直接的依赖，做到了所有系统服务并发启动。这一设计同样是 Systemd 独具特色的创意。对于用户定义的服务，Systemd 允许配置其启动依赖的项目，从而保证服务必须按照必要的顺序运行。

#### 跟踪和管理进程的生命周期

在 Systemd 之前的主流应用管理服务都是使用 “进程树” 来跟踪应用的继承关系，而进程的父子关系很容易通过 `两次 fork` 的方法来脱离。

而 Systemd 则提出通过 CGroup 跟踪进程关系，弥补了这个缺漏。通过 CGroup 不仅能够实现服务之间的访问隔离，限制特定应用程序对系统资源的防伪配额，还能更精确的管理服务的生命周期。

#### 统一管理服务日志

Systemd 是一系列工具的集合，包括了一个专用的系统日志管理服务：Journald。这个服务的设计初衷是克服现有 Syslog 服务的日志内容易伪造和日志格式不统一等缺点，现在已经纳入了 Systemd 的标准姊妹服务中。Journald 用二进制格式保存所有日志信息，因而日志内容很难被手工伪造。Journald 还提供了一个 journalctl 命令来查看日志信息，这样使得不同服务输出的日志具有相同的排版格式，便于数据的二次处理。



### Unit 和 Target

Unit 是 Systemd 管理服务的基本单元，可以认为每个服务就是一个 Unit，并使用一个 Unit 文件定义。在 Unit 文件中需要包含相应服务的描述、属性以及需要运行服务的命令。

Systemd 主要又如下单元类型：

| 类型       | 描述                          |
| ---------- | ----------------------------- |
| .service   | 系统服务                      |
| .mount     | 挂载点                        |
| .automount | 自动挂载点                    |
| .socket    | 套接字                        |
| .device    | 系统设备                      |
| .swap      | 交换分区                      |
| .path      | 文件路径                      |
| .target    | 启动目标                      |
| .timer     | 计时器                        |
| .scope     | 不是由 Systemd 启动的外部进程 |
| .slice     | 进程组                        |
| .snapshot  | Systemd 快照                  |



Target 是 Systemd 中用于指定服务组的启动方式，相当于 SysV-init 中的 `运行级别`。每次系统启动时都会运行与当前系统具有同级别 Target 关联的所有服务，如果服务不需要跟随系统自动启动，则完全可以忽略这个 Target 的内容。通常来说，大多数 Linux 用户平时使用的都是 `多用户模式` 这个级别，对应的 Target 值为 `multi-user.target`，它又一个等效的可用值是 `default.target`。

```shell
# 查看当前系统的所有 Target
$ systemctl list-unit-files --type=target

# 查看一个 Target 包含的所有 Unit
$ systemctl list-dependencies multi-user.target

# 查看启动时的默认 Target
$ systemctl get-default

# 设置启动时的默认 Target
$ sudo systemctl set-default multi-user.target

# 切换 Target 时，默认不关闭前一个 Target 启动的进程，
# systemctl isolate 命令改变这种行为，
# 关闭前一个 Target 里面所有不属于后一个 Target 的进程
$ sudo systemctl isolate multi-user.target
```



Systemd 使用 Target 取代了 System V Init 的运行级的概念。

| 运行级别     | Systemd 目标                                          | 备注                                                      |
| ------------ | ----------------------------------------------------- | --------------------------------------------------------- |
| 0            | runlevel0.target, poweroff.target                     | 关闭系统                                                  |
| 1, s, single | runlevel1.target, rescue.target                       | 单用户模式                                                |
| 2, 4         | runlevel2.target, runlevel4.target, multi-user.target | 用户定义/域特定运行级别。默认等同于 3                     |
| 3            | runlevel3.target, multi-user.target                   | 多用户，非图形化。用户可以通过多个控制台或网络登录        |
| 5            | runlevel5.target, graphical.target                    | 多用户，图形化。通常为所有运行级别 3 的服务外加图形化登录 |
| 6            | runlevel6.target, reboot.target                       | 重启                                                      |
| emergency    | emergency.target                                      | 紧急 Shell                                                |



### 日志管理

Systemd 统一管理所有 Unit 的启动日志。带来的好处就是，可以只用 `journalctl` 一个命令，查看所有日志（内核日志和应用日志）。日志的配置文件是 `/etc/systemd/journald.conf`。

```shell
# 查看所有日志（默认情况下 ，只保存本次启动的日志）
$ sudo journalctl

# 查看内核日志（不显示应用日志）
$ sudo journalctl -k

# 查看系统本次启动的日志
$ sudo journalctl -b
$ sudo journalctl -b -0

# 查看上一次启动的日志（需更改设置）
$ sudo journalctl -b -1

# 查看指定时间的日志
$ sudo journalctl --since="2012-10-30 18:17:16"
$ sudo journalctl --since "20 min ago"
$ sudo journalctl --since yesterday
$ sudo journalctl --since "2015-01-10" --until "2015-01-11 03:00"
$ sudo journalctl --since 09:00 --until "1 hour ago"

# 显示尾部的最新10行日志
$ sudo journalctl -n

# 显示尾部指定行数的日志
$ sudo journalctl -n 20

# 实时滚动显示最新日志
$ sudo journalctl -f

# 查看指定服务的日志
$ sudo journalctl /usr/lib/systemd/systemd

# 查看指定进程的日志
$ sudo journalctl _PID=1

# 查看某个路径的脚本的日志
$ sudo journalctl /usr/bin/bash

# 查看指定用户的日志
$ sudo journalctl _UID=33 --since today

# 查看某个 Unit 的日志
$ sudo journalctl -u nginx.service
$ sudo journalctl -u nginx.service --since today

# 实时滚动显示某个 Unit 的最新日志
$ sudo journalctl -u nginx.service -f

# 合并显示多个 Unit 的日志
$ journalctl -u nginx.service -u php-fpm.service --since today

# 查看指定优先级（及其以上级别）的日志，共有8级
# 0: emerg
# 1: alert
# 2: crit
# 3: err
# 4: warning
# 5: notice
# 6: info
# 7: debug
$ sudo journalctl -p err -b

# 日志默认分页输出，--no-pager 改为正常的标准输出
$ sudo journalctl --no-pager

# 以 JSON 格式（单行）输出
$ sudo journalctl -b -u nginx.service -o json

# 以 JSON 格式（多行）输出，可读性更好
$ sudo journalctl -b -u nginx.serviceqq
 -o json-pretty

# 显示日志占据的硬盘空间
$ sudo journalctl --disk-usage

# 指定日志文件占据的最大空间
$ sudo journalctl --vacuum-size=1G

# 指定日志文件保存多久
$ sudo journalctl --vacuum-time=1years
```



### 单元文件存放路径

```shell
$ sudo systemctl show --property=UnitPath

# 按优先级从低到高显示加载目录
UnitPath=/etc/systemd/system /run/systemd/system /run/systemd/generator /usr/local/lib/systemd/system /usr/lib/systemd/system /run/systemd/generator.late
```

- /usr/lib/systemd/system/：软件包安装的单元
- /etc/systemd/system/：系统管理员安装的单元



### 命令汇总

```shell
#重载单元
$ sudo systemctl daemon-reload

#立即激活单元
$ sudo systemctl start UNIT

#立即停止单元
$ sudo systemctl stop UNIT

#立即重启单元
$ sudo systemctl restart UNIT

#重新加载配置
$ sudo systemctl reload UNIT

#输出单元运行状态
$ sudo systemctl status UNIT

#检查单元是否配置为自动启动
$ sudo systemctl is-enabled UNIT

#开机自动激活单元，根据[Install]设置的 WantedBy 会在 /etc/systemd/system/TARGET.wants/ 目录中创建软链
$ sudo systemctl enable UNIT

#设置单元为自动启动并立即启动这个单元
$ sudo systemctl enable --now UNIT

#取消开机自动激活单元
$ sudo systemctl disable UNIT

#禁用一个单元（禁用后，间接启动也是不可能的）
$ sudo systemctl mask UNIT

#取消禁用一个单元
$ sudo systemctl unmask UNIT

#显示单元的手册页（必须由单元文件提供）
$ sudo systemctl help UNIT

#列出一个 Unit 的所有依赖，默认不会展开显示。如果要展开 Target，就需要使用--all参数。
$ sudo systemctl list-dependencies [--all] UNIT

#列出所有配置文件
$ sudo systemctl list-unit-files

#列出指定类型的配置文件
$ sudo systemctl list-unit-files --type=service
```



### 电源管理命令

```
#重启
$ sudo systemctl reboot

#退出系统并关闭电源
$ sudo systemctl poweroff

#待机
$ sudo systemctl suspend

#休眠
$ sudo systemctl hibernate

#混合休眠模式（同时休眠到硬盘并待机）
$ sudo systemctl hybrid-sleep
```



# 参考

[Weikipedia - UNIX System V](https://zh.wikipedia.org/wiki/UNIX_System_V)

[archlinux - init](https://wiki.archlinuxcn.org/wiki/Init)

[archlinux - SysVinit](https://wiki.archlinuxcn.org/zh/SysVinit)

[可能是史上最全面易懂的 Systemd 服务管理教程！](https://cloud.tencent.com/developer/article/1516125)

[linux的systemd与init的区别](https://bbs.huaweicloud.com/blogs/218845)

[Linux启动流程和服务管理(init和systemd)](https://blog.51cto.com/csnd/5482899)

[浅析 Linux 初始化 init 系统：sysvinit](https://www.cnblogs.com/liyuanhong/articles/4344186.html)

[深入了解 Systemd 之概要介绍](https://itlanyan.com/deep-inside-systemd-brief/)

