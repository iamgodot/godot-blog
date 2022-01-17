---
title: "From BitTorrent to Firewall"
date: 2021-12-20T17:00:00+08:00
categories:
  - Code
tags:
  - network
draft: false
---

服务器能做什么？在 [Awesome-Selfhosted](https://github.com/awesome-selfhosted/awesome-selfhosted) 里可以找到上百种答案。如果带宽不算太小的话，那么 BT 下载是个不错的尝试。借着 [No Time to Die](https://movie.douban.com/subject/20276229/) 的上映我开始重温 007 系列，从皇家赌场到幽灵党，在服务器上的下载体验是很好的。

# BitTorrent

在此之前，我基本上把 BT、种子、磁力、迅雷下载当成同一种东西。下载电影？先找种子或者磁力链接，打开迅雷下载，然后视速度决定要不要开个会员。

实际上这完全曲解了 BT 下载。

首先 BitTorrent 是一种网络协议。还记得计算机网络一开始就提到过除了 C/S 架构之外，还有 P2P(Peer-to-peer)，也就是网络中的各个节点都扮演了同等的角色，既是客户端也是服务器。BT 基于 P2P 实现了去中心化的文件分享，让网络数据的传输不再仅限于服务器的能力，而是共享带宽，每个人下载的同时也在上传，所以越多人参与速度就越快。

类似于 HTTP 和 FTP，BT 也是基于 TCP/IP 的一种应用层协议。基本上它是这么运作的：

- 我有一部电影，想把资源分享到网络，要先提供一个种子文件

- 种子文件实际上就是个文本文件，里面主要记录两部分信息
  - Trackers: 就是 Tracker 服务器的地址，这个服务器不是用来下载资源的，而是用于获取其他 Peers 的联系方式
  - Files: 一个视频文件会被（逻辑）划分为很多个虚拟分块，每块的索引和验证码都包含在这里
- 接下来我把种子文件发布出去，等待别人下载
- 这时候有人获取到种子了，于是开启了 BT 客户端下载
- 客户端先解析种子文件中的信息，找到 Tracker，然后询问有哪些 Peers
- 因为是第一个下载者，所以 Tracker 告知 Peer 目前只有我，也就是发布者
- 之后对方会尝试与我互连，然后根据 Files 信息交换数据，这里基本就是我上传给对方
- 下载的过程会以分块为单位进行，每块完成下载后会根据验证码再做校验
- 如果这时又有一个人开始下载，那么我和这第一个下载者都会贡献上传
- 随着更多用户的参与，（上传）下载的速度就会越来越快

可以发现，整个过程中 Tracker 是很关键的一步，如果没有有效的 Tracker 提供 Peers，后面的下载都无法开始。所以如果你的 BT 下载没有速度，首先要尝试多添加一些 Tracker 服务器，比如 [TrackersList](https://trackerslist.com/#/zh).

为了避免 Tracker 成为瓶颈，又出现了 DHT(Distributed Hash Table) 来帮助 Peers 的寻找：

> 在不需要服务器的情况下，每个客户端负责一个小范围的路由，并负责存储一小部分数据，从而实现整个 DHT 网络的寻址和存储。使用支持该技术的 BT 下载软件，用户无需连上 Tracker 就可以下载，因为软件会在 DHT 网络中寻找下载同一文件的其他用户并与之通讯，开始下载任务。

这一切看起来都很（过于）理想：人人为我，我为人人。但是事实从来没有如此美好。

# Breaking Rules

首先在国内的网络环境中，BT 的使用就已经打折扣了：

- 家庭网络一般都没有公网 IP
- 上传速度远小于下载速度（对带宽经济，但对于 BT 并不是好事）
- ISP 的干扰，比如针对 BT 数据包的干扰

其次就是流氓下载软件野蛮地改变了 BT 平等分享的初衷，比如迅雷。那么迅雷是如何打破规则的呢？简单来说，它在下载的时候老老实实享受其他用户的上传，等到分享时却只提供资源给自家用户，俗称吸血。不仅如此，迅雷还会维护单独的服务器，这样下载既 From peer 又 From server，也就是 P2SP，速度自然超过普通 BT 软件，同时还让人感觉很多资源只有迅雷才下得动。至于会员专享什么的，把一开始设的限制（比如连接其他 Peer 的数量）解除掉就好了，剩下的就是大肆捞钱。

国内类似的吸血下载软件不少，但基本以迅雷为首。这么多年积累下来，就算大家觉得过分可能也无奈了，不用迅雷实在是下载不了啊。话虽如此，该抵制的还是要抵制，至少我已经从（多年前的）迅雷会员变为 BT 共产主义者了。

最后，很多人使用 BT 也常常不遵守基本原则，比如关闭上传、下载完之后立刻删除种子资源。这也导致了很多软件增加了针对不良使用习惯的用户的限制，还出现了 PT(Private Tracker) 下载，简单讲就是在一个私密的圈子内分享资源下载，保证每个 Peer 的上传贡献。

一个良好的网络环境既离不开硬性限制，也需要每个人的自我约束。更重要的前提是，打破黑盒，大家了解到事情背后的真相才有可能做出正确的决定。

# Qbittorrent

抛弃了迅雷怎么办？其实好的选择有很多，比如 Windows 上的 [IDM](https://www.internetdownloadmanager.com/)，或者开源的 Motrix, qbittorrent 等等。

对于 Linux 来说，本地使用我觉得 Motrix 很不错，支持 AppImage，界面好看，还能自动更新各种 TrackerList，总之就是省心。而服务器部署我选择了 qbittorrent，GitHub 上面的文档很详尽，用 Docker 部署非常简单。

这里要提一下，一开始使用最好先下个热门资源确认 BT 可以正常工作，比如 [Ubuntu 的镜像文件](https://releases.ubuntu.com/20.10/ubuntu-20.10-desktop-amd64.iso.torrent)。

说回 qbittorrent，虽然不像 Motrix 可以自动更新 Trackers，但是它集成了种子的搜索功能（基本是国外资源），还有强大的 RSS 订阅。

我的服务器是 5M 带宽，但腾讯云的出入网计算方式不太一样，对于入网速度，小于 5M 的带宽都按照 10M 计算。即便如此，下载速度还是超过了应有的限制，也许还有服务器的网络质量加成吧。

# Firewall

下着下着我感到很奇怪，因为 qbt 是用到了几个端口的，比如 UI 和下载，而下载的 6881 端口并没有在安全组开放啊，为什么还能正常使用？

先用 nmap 测试下：

```bash
nmap -p 6881 server
```

显示端口 filtered，确实是被防火墙拦住了没错。那是怎么做到的，BT 可以穿越防火墙？

一番搜索之后我找到了答案，主要还是不太了解防火墙的工作机制：

- 首先如果设置了端口禁止入网，那么外网肯定是无法向端口开启连接的
- 但是如果端口可以出网就不一定了，这时端口可以主动建立连接
- 之后外面再返回的数据包就可以穿过防火墙了

以上针对的是 Stateful firewall 的情况，也就是防火墙会跟踪并维护所有的网络连接，如果一个连接已经建立，那么即使规则不允许端口入网，数据仍然是可以正常传递的。

再看下[腾讯云的安全组介绍](https://cloud.tencent.com/document/product/213/12452)：

> 安全组是一种虚拟防火墙，具备有状态的数据包过滤功能，...

果然是有状态的。

不如再做个实验好了。因为家里没有公网，所以又临时开了台 vultr 服务器。先在腾讯云上监听端口：

```bash
nc -l -p 6881
```

然后从 vultr 尝试连接：

```bash
nc tecent_server_ip 6881
```

显示超时，因为端口被 filtered，没问题。再在 vultr 上监听：

```bash
nc -l -p 8888
```

从腾讯云主动发起连接：

```bash
nc -p 6881 vultr_server_ip 8888
```

果然双向可以正常通信。所以 qbittorrent 才能正常下载啊。

---

*References*

- [BitTorrent - Wikipedia](https://zh.wikipedia.org/wiki/BitTorrent_(%E5%8D%8F%E8%AE%AE))
- [PT 下载 - Wikipedia](https://zh.wikipedia.org/wiki/PT%E4%B8%8B%E8%BC%89)

- [P2SP - Wikipedia](https://zh.wikipedia.org/wiki/P2SP)
- [Stateful firewall](https://en.wikipedia.org/wiki/Stateful_firewall)
- [Motrix](https://motrix.app/)
- [Qbittorrent - GitHub](https://github.com/qbittorrent/qBittorrent)
- [为什么国内 BT 环境如此恶劣？ - 知乎](https://zhuanlan.zhihu.com/p/87193566)
