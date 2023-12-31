---
title: "From Wireshark to Linux Capabilities"
date: 2021-11-21T13:39:25+08:00
categories:
  - Code
tags:
  - linux
draft: true
---

Tcpdump 和 Wireshark 是抓包必备的程序，但是由于需要截取网络数据包，所以在 Linux 下必须以 root 的身份来运行。每次都 sudo 执行不方便也并不安全（对 Wireshark 来说捉包只是一小部分功能），解决方案当然有，在寻找的过程中我了解到了 Capabilities 的冰山一角。

# TL;DR

- 可以通过设置 Setuid 以 root 身份执行，但如此以来赋予了过高的权限（也没有必要）。
- Linux 下用 Capabilities 把系统权限划分成多个条目，以此实现细粒度地提升程序的执行能力。

# Setuid

先盗一张图复习下 Linux File Permission 的基础知识：

![](https://static.iamgodot.com/content/images/fig_permissions_chmod-command.jpg)

除了 rwx 之外还存在三种特殊类型，即是为了在更高权限下运行程序：

- Setuid: 程序的运行者不再是执行者，而是变成文件的所有者
- Setgid: 程序的运行群组变成了文件的所在群组，如果给目录设置，那么其中新建的文件所有群组会变成目录的群组而不是执行者的群组
- Sticky bit: 针对目录设置，目录下的文件只有所有者和 root 能够重命名、移动和删除

在 Linux 中 sudo 是 Setuid 最好的例子，而 crontab 和 tmp/ 分别是 Setgid 和 Sticky bit 的典型应用。

在命令行中测试：

```shell
# Setuid
➜  ~ umask -S
u=rwx,g=rx,o=rx
➜  ~ umask  # 掩码是 022，所以默认文件的权限是 666-022=644，而目录则是 777-022=755
022
➜  ~ touch foo.bar
➜  ~ l foo.bar
-rw-r--r-- 1 godot godot 0 Nov 21 14:40 foo.bar
➜  ~ chmod u+s foo.bar  # 也可以使用 chmod 4644 来 setuid
➜  ~ l foo.bar  # 之所以 S 会大写是因为还没有赋予 execute 权限
-rwSr--r-- 1 godot godot 0 Nov 21 14:40 foo.bar
➜  ~ chmod u+x foo.bar
➜  ~ l foo.bar  # 此时 s 变为正常小写
-rwsr--r-- 1 godot godot 0 Nov 21 14:40 foo.bar

➜  ~ chmod g+xs foo.bar  # 可以使用 chmod 2644 来 setgid
➜  ~ l foo.bar
-rwsr-sr-- 1 godot godot 0 Nov 21 14:48 foo.bar

➜  ~ mkdir foobar
➜  ~ l | grep foobar
drwxr-xr-x  2 godot godot 4.0K Nov 21 14:50 foobar
➜  ~ chmod +t foobar  # 可以使用 chmod 1755 来 set sticky bit
➜  ~ l | grep foobar
drwxr-xr-t  2 godot godot 4.0K Nov 21 14:50 foobar
```

在 Tcpdump 上试试：

```shell
➜  ~ l $(which tcpdump)
-rwxr-xr-x 1 root root 1.3M Jun 10 18:46 /usr/bin/tcpdump
➜  ~ tcpdump
tcpdump: wlp0s20f3: You don't have permission to capture on that device
(socket: Operation not permitted)
➜  ~ sudo chmod u+s /usr/bin/tcpdump
➜  ~ tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on wlp0s20f3, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```

试验成功，Wireshark 是用 `/usr/bin/dumpcap` 来捉包的，所以用同样的方法 Setuid 即可。

注意不要用 Shell Script 去测试 Setuid，因为 Linux 会忽略 Shebang 类的可执行文件，比如 `#!/bin/sh` 在文件开头的脚本。这会带来系统性的漏洞风险，参考 [Allow-setuid-on-shell-scripts](https://unix.stackexchange.com/questions/364/allow-setuid-on-shell-scripts).

# Capabilities

从 [Wireshark 的文档](https://gitlab.com/wireshark/wireshark/-/wikis/CaptureSetup/CapturePrivileges#most-unixes)看到 Setuid 和 Setcap 都可以实现 Privilege Separation，也就是 GUI 的部分仍然按照普通用户的身份执行，而 Dumpcap 在更高的权限上运行。

因为 Setuid 给予的权限过大，所以 Setcap 其实是更好的选择：

```bash
# 需要确认 Wireshark 使用的是 /usr/bin/dumpcap 还是 /usr/sbin/dumpcap
sudo setcap cap_net_raw,cap_net_admin+eip /usr/bin/dumpcap
```

那么 Capabilities 到底是什么？根据 [Arch Wiki](https://wiki.archlinux.org/title/capabilities):

> Capabilities provide fine-grained control over superuser permissions, allowing use of the root user to be avoided. Software developers are encouraged to replace uses of the powerful setuid attribute in a system binary with a more minimal set of capabilities. Many packages make use of capabilities, such as CAP_NET_RAW being used for the ping binary provided by iputils. This enables e.g. ping to be run by a normal user (as with the setuid method), while at the same time limiting the security consequences of a potential vulnerability in ping.

不过本地执行了下 `getcap /usr/bin/ping` 并没有看到相应的权限，但命令仍然是能正常使用的。

Capabilities 实际上是通过 [xattr(Extended Attributes)](https://wiki.archlinux.org/title/File_permissions_and_attributes#Extended_attributes) 来实现的，这是一种给文件增加额外属性的方式，所有的属性分为 Security, System, Trusted, User 四种，随便实验一下：

```shell
➜  ~ touch foo.bar
➜  ~ setfattr -n user.foo -v bar foo.bar
➜  ~ getfattr -d foo.bar
# file: foo.bar
user.foo="bar"

➜  ~ setfattr -x user.foo foo.bar  # 删除属性
```

注意 Extended Attributes 不是默认被 cp 和 rsync 等程序保留的，所以使用的时候需要确认相关支持。比如 Capabilities 在使用 `cp -a` 的时候会自动复制，但是对于 `rsync` 来说则需要额外选项：`rsync -X`.

在 Capabilities 之前，Linux 执行程序的权限分为 root 和非 root 两种，而现在系统的特权被分为不同的功能组，执行线程（进程）的时候只需要去检查组中的权限是否足够执行某个命令。

[Capabilities](https://man.archlinux.org/man/capabilities.7) 相关的权限列表很长，我们要用到的有：

- CAP_NET_RAW: 允许使用原生 Socket
- CAP_NET_ADMIN: 允许执行网络相关操作

对文件来说，有三个集合来存放权限（Wireshark 使用 Setcap 就是把两个网络权限条目加入到 PIE 三个集合中）:

- Permitted
- Inheritable
- Effective

进程还有两个额外的集合：

- Bounding
- Ambient

在调用 [execve](https://linux.die.net/man/2/execve) 方法执行程序时，内核会根据文件和进程的权限集合计算出最终的集合结果，然后判断权限是否足够执行。

具体的计算逻辑比较复杂，参考[这篇](https://cloud.tencent.com/developer/article/1529342)，等需要用的时候再继续研究。

---

*References*

- [Wireshark - Archwiki](https://wiki.archlinux.org/title/wireshark)
- [Capabilities - Archwiki](https://wiki.archlinux.org/title/capabilities)
- [Extended Attributes - Archwiki](https://wiki.archlinux.org/title/File_permissions_and_attributes#Extended_attributes)
