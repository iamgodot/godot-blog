---
title: "关于 CORS"
date: 2022-05-17T23:42:11+08:00
draft: true
categories:
  - Code
tags:
  - HTTP
keywords:
  - CORS 解决
  - 跨域资源共享
---

说起 CORS，就不得不先提到 SOP(Same-origin policy)：浏览器打开的网页只可以对该网页的同源网站发起请求。注意，受约束的主要是脚本代码，不包括图片或者 CSS 等资源（字体文件是个例外）。同源的定义包括三部分，即协议、域名和端口都要保持一致。

为了缓解 SOP 带来的严格限制，有几种主流的解决方案可以选择：

1. CORS
2. JSONP：利用 `<script>` 标签来请求非同源地址的 JSON 响应，同时配合一个预先定义的回调函数来处理响应数据。
3. WebSocket：WS 连接并不受同源策略的约束，但是在建立连接时服务端也需要判断 headers 中的 `Origin` 是否可以接受。

其中 CORS 应该是最实用的一种，相比 JSONP 只支持 GET 请求，前者扩展了各种 HTTP 方法的跨域调用。

CORS(Cross-origin resource sharing)，是一种跨域共享资源的机制，它利用特定的 Headers 来保证跨域请求的安全性，这些请求分为两类：简单请求和非简单请求。

简单请求，包括 GET、HEAD 和 POST，这里 POST 的 Content-Type 仅限于下面三种：

- `application/x-www-form-urlencoded`
- `multipart/form-data`

- `text/plain`

对于这些请求来说，只需要保证 `Access-Control-Allow-Origin` 中匹配了当前网页的域名即可，如果是 `*` 的话表明所有的域名都是允许的。

非简单请求，比如 Content-Type 为 `application/json` 的 POST，会增加一次额外的 Preflight 请求，即先发送 OPTIONS 请求给服务器，然后通过响应中的一系列 Headers 决定是否可以进行真正的请求。这些 Headers 包括：

- `Access-Control-Allow-Methods`：服务器允许的跨域方法，比如 POST。
- `Access-Control-Allow-Headers`：服务器允许的跨域头部，比如 Content-Type。
- `Access-Control-Max-Age`：Preflight 请求结果的缓存时间，默认为 5s。

另外，如果想在 Chrome 中查看 Preflight 请求的话，打开 Network 标签，点击 Other filter 就可以看到了。

# Credential

对于用到 Cookie 或者其他 HTTP Auth 信息的请求，还需要通信双方做一些额外的设置，服务器要提供：

- `Access-Control-Allow-Credentials`：设置为 true。在 Preflight 请求的响应中，这个 Header 表示正式请求中可以携带 Credential 信息。而对于 GET 这种不需要 Preflight 的简单请求，没有此 Header 的话浏览器会直接拒绝服务器的响应。
- `Access-Control-Expose-Headers`：允许设置的 Headers 暴露给客户端。对于 Cookie 场景，要添加 `Set-Cookie`，浏览器脚本才可以使用此 Header，进而 Cookie 才能够设置成功。

而客户端想发送 Credential 请求的话，也要显式地加上一个特殊的 flag： `withCredentials: true`，无论是 XMLHttpRequest、Fetch 还是 Axios。

最后要注意的是，对于 Credential 请求的响应，下面三种 Headers 不可以使用 `*` 匹配，必须要指定特定的合法值：

- `Access-Control-Allow-Origin`
- `Access-Control-Allow-Headers`
- `Access-Control-Allow-Methods`

# CORS in Python

使用 Flask 的话，可以安装 [Flask-CORS](https://flask-cors.readthedocs.io/en/latest/) 来实现服务端的跨域：

```python
from flask import Flask
from flask_cors import CORS

app = Flask(__name__)
CORS(app, supports_credentials=True, allow_headers=['Content-Type'], expose_headers=["Set-Cookie"])
```

如果是 FastAPI，直接用内置的 Middleware 就好了，参考[官方文档](https://fastapi.tiangolo.com/tutorial/cors/)。

# Proxy

其实换一种思路的话，只要有中间网关做转发来避免跨域请求就不存在 SOP 的限制了。

所以在前端开发中，比如 React，只需配置 proxy 为服务端的地址，API 请求就会先传递到 Dev server，再转发给后端。

同理，在正式部署中，我们同样可以利用 Nginx 的 `proxy_pass` 达到代理转发的效果。

当然，即便如此，了解 SOP 和 CORS 的原理依然是必要的，而且也并不困难。

---

*References*

- [CORS - MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)

- [Same-origin policy - Wikipedia](https://en.wikipedia.org/wiki/Same-origin_policy)
- [Cross-origin resource sharing - Wikipedia](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing)
- [Proxying API Requests in Development - React](https://create-react-app.dev/docs/proxying-api-requests-in-development/)
