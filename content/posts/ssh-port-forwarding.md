---
title: "About SSH Port Forwarding"
date: 2021-12-22T11:36:49+08:00
categories:
  - Code
draft: false
---

SSH 的端口转发很实用，但我总觉得难以理解和记忆，直到最近才有所好转。

主要原因还是发现了更好的使用场景。以前基本就是拿来做内网穿透，现在想想，最佳应用反而是用来绕过服务器的防火墙，大多数的端口虽然都是被禁用的（至少禁止入网，这也是正常的安全措施），但是想要连接的话直接本地端口转发就可以实现了。

# Local Port Forwarding

为什么叫做本地呢，我想有两个原因：

- 转发的端口在当前（执行 SSH 命令这台）机器上
- 请求是从当前机器发出的

当前机器就是我的笔记本，另外一台是服务器。比如，在服务器上部署一个应用，开放给 8000 端口，但是被墙掉了，没办法在本地调试，怎么办？防火墙肯定开放了 SSH 登录的端口，比如 22，那么就让请求从本地的端口发送到服务器的 22 端口，再转发到 8000 端口，最后原路返回。我可以设置本地的端口也是 8000，这样直接用 localhost:8000 来访问应用就好了。

转发的重点在于本地的 8000 端口和服务器的 22 端口之间，因为请求到了服务器之后可以给应用的 8000，也可以给其他的机器，只要服务器能连接到：

```bash
# ssh -L local_port:dest_addr:dest_port server

# Local 8000 < -- > Server 22 < -- > Server 8000
# -fNT 让 ssh 不要打开服务器 shell，并且转为后台运行
# server 隐含了使用 22 端口登录，当然也可以在 ssh config 中设置任意登录端口
ssh -fNT -L 8000:localhost:8000 server
```

注意这里的 dest 对应的 src 是 server，也就是说 localhost 及后面的 8000 都是 server 的 IP 和端口。可以理解为 server 是中介，整条通路是 local -> server -> dest.

除了调试应用，还可以做各种各样的事情，比如浏览文件（在服务器上开个 `python -m http.server 8080`），用 netcat 测试连接，甚至实现流媒体播放。最重要的是，这解放了封闭在防火墙背后的一系列端口。

不光如此，如果服务器还连接到其他的机器，就可以将三台机器联系到一起。跳板机跨越是一个常见的例子，即本地先 SSH 登录到跳板机，再登录到服务器。利用端口转发的话只需要将 dest 设置为跳板机的地址和端口（当然，直接使用 SSH 的 ProxyJump 更加直截了当，这里有些大材小用了）。

本地端口转发很好，但不是万能的。如果服务器没有公网 IP，或者隐藏在 NAT 背后呢？这时候就需要远程端口转发。

# Remote Port Forwarding

先解释一下我对远程两个字的理解：

- 转发的端口在另外一台机器上
- 请求是从另外一台机器发出的

接着刚才的例子，如果服务器躲在 NAT 后面不出来，但是我的笔记本可以通过 SSH 连接，那就可以在服务器上设置远程端口转发：

```bash
# ssh -qngfNT -R [bind_addr]:remote_port:dest_addr:dest_port host

ssh -qngfNT -R 8000:localhost:8000 host
```

这里的 host 指的就是我的笔记本，所以服务器要可以 SSH 到我的笔记本才行？是的，这些转发的可能性都在于，必须能建立起双方之间的 SSH 会话，这也是为什么端口转发又被称为 SSH 隧道，有了通路，才可以换方向。

和本地转发不同的是，dest 的 src 是 local，也就是当前机器（服务器），通路是 local -> host -> dest.

其实远程端口转发最常见的用法是实现内网穿透，也是解决 NAT 的问题。比如公司的电脑在 NAT 后，通过公网服务器，我可以从家里登录到公司。

在这个场景中，首先要保证几个前提：公司电脑和服务器都要开启了 SSH 的服务，即都能够登录上去；其次，如果想要用家里的电脑，即第三台机器，实现公司登录的话，要设置服务器 SSH 服务的 GatewayPorts 选项为 yes. 如果这些都满足，那么直接在公司电脑执行：

```bash
# -qngfNT 一堆不是很重要但最好加上的选项
ssh -qngfNT -R 8000:localhost:22 server
```

8000 就是（公司电脑远程转发给）服务器的端口，在家里用这个端口登录服务器即可开始办公：

```bash
ssh user-company@ip-server -p 8000
```

注意上面示例中的 `bind_addr`，如果想只让某个固定 IP 才能登录，那么在这里加上就可以了。当然这要求家里的电脑有公网地址，而且服务器上的 GatewayPorts 设置为 clientspecified.

OK，现在我们可以进入公司网络了，但如果公司电脑（或者办公网络）中有个需要调试的应用怎么办？那就再加一个本地端口转发：

```bash
ssh -fNT -L 8000:localhost:8000 server
```

现在就能用 localhost:8000 访问公司的应用了，那么如果有多个应用呢？不管是 -L 还是 -R 都支持多个参数，所以分别用不同的端口指定即可。

---

*References*

- [SSH port forwarding](https://www.ssh.com/academy/ssh/tunneling/example)

- [远程操作与端口转发 - 阮一峰](http://www.ruanyifeng.com/blog/2011/12/ssh_port_forwarding.html)
- [一个很好的内网穿透的例子](http://arondight.me/2016/02/17/%E4%BD%BF%E7%94%A8SSH%E5%8F%8D%E5%90%91%E9%9A%A7%E9%81%93%E8%BF%9B%E8%A1%8C%E5%86%85%E7%BD%91%E7%A9%BF%E9%80%8F/)