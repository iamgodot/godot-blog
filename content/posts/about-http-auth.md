---
title: "关于 HTTP Auth"
date: 2022-01-28T10:16:19+08:00
categories:
  - Code
tags:
  - HTTP
draft: true
keywords:
  - HTTP 认证
  - OAuth 认证
  - jwt 认证
  - Digest 认证
---

Auth 代表了 Authentication 和 Authorization 两个概念，也就是认证与授权。基于 HTTP，两者得以遵循一定的标准，SSL/TLS 之后，又出现了 OAuth 2.0，让授权也简单了许多。

# Authentication

认证相对来说比较直接，核心就是对 Credential(e.g. username/password) 的验证。HTTP 提供了多种认证方案，比如最常见的 Basic auth, Digest access 和 Bearer.

## Basic auth

具体来说就是服务器用 `WWW-Authenticate` 表示需要认证，比如 `WWW-Authenticate: Basic realm='Accessing to xx site'`，客户端则通过 `Authorization` 提供相关信息：`Authorization: Basic Zm9vOmJhcg==`，后面的一串编码是对用户名密码明文进行 base64 的结果，即可以直接从中 decode 出原始信息 `foo:bar`. 没有 HTTPS 的保护，这样很不安全，所以 Apache/Nginx 对 BA 的实现都会使用密码的哈希结果而不是原文，拿后者举例：

```nginx
http {
    server {
        location / {
            auth_basic "Accessing to xx site";
            auth_basic_user_file /path/to/authfile;
        }
    }
}
```

然后需要在 `authfile` 中保存 username/password pair，比如 `sudo htpasswd -c /path/to/authfile user1`，`htpasswd` 是 Apache 提供的专门用来生成 BA 使用的 Credential file 的工具。不用额外安装，我们直接用 `openssl` 代替：

```shell
$ openssl passwd -apr1 foobar
$apr1$BICE9LF.$9wYc.j1KyU/2/UtBg8l2W.

$ openssl passwd -apr1 -salt BICE9LF.
$apr1$BICE9LF.$9wYc.j1KyU/2/UtBg8l2W.
```

第一行的 `apr1` 表示使用 Apache 变种的 MD5 哈希算法，生成的结果被 `$` 分隔为三部分，第二部分是随机生成的 salt，最后是哈希结果。所以第二行里，如果我们用同样的 salt 去生成就会得到相同的哈希值。

## Digest access

BA 在简单的场景下配合 HTTPS 还是可以用的，哈希则给密码保护提供了新的校验思路，比如用户只需要传递密码的哈希值，再比如服务端不保存密码明文。哈希的好处包括单向（防止泄露）、简短（做摘要、签名）和变化（加 salt 可以有效对抗彩虹表），而服务器只要比较计算结果是否一致就达到了校验的目的，这样就形成了 HTTP 的 Direct Access 机制：服务端提供 nonce 和 qop(quality of protection) 参数给客户端，客户端按需生成结果返回。

```
HA1 = MD5(username:realm:password)
HA2 = MD5(method:digestURI)
response = MD5(HA1:nonce:HA2)
```

根据参数的不同，HA 也要调整，比如如果 qop 指定了 `auth-init`，那么 HA2 就要变成 `HA2 = MD5(method:digestURI:MD5(entityBody))`. Header 中也要体现 Digest 的使用，比如客户端会返回 `Authorization: Digest username='...', realm='...', response='...' ...`. 总的来说，DA 的方式不需要 HTTPS 也能够有效地保护密码安全，但是仍然阻挡不了中间人的攻击方式，因为客户端无法确认服务端的真正身份。

## 调用认证

上面说的是登录认证，此外还有调用认证，比如 API. 对于泄露、篡改和伪装这三大安全问题，调用认证明显更需要防范篡改的攻击方式，此时签名技术就出现了：MAC(Message Authentication Code)，也就是对消息做摘要，以此保证其中的内容没有改动。哈希自然很适合这项工作，所以 HMAC(Hash-based MAC) 也出来了。

![](https://static.iamgodot.com/content/images/20220128172925.png)

MAC 有个前提是通信双方共享了某个 Secret key，而在 HMAC 相当于 Salted hashing，其中的 Salt 就是用到了 Secret key. 也就是说这类似于对称加密，服务器和用户在通信之前需要先商量好密钥。举例来说，这相当于你先注册或登录 AWS/阿里云/腾讯云的帐号，在里面创建了子帐户，此时你会得到密钥对 AppID/AppSecretKey，这个 AppSecretKey 就是给 HMAC 用的，而 AppID 标识了密钥在你的帐号下面。这样一来，权限的粒度也通过密钥对变得更小了，一个帐号下面可以开通多个 App，每个都对应不同的权限范围。

也许 TLS 之前这种实现带来的安全保障是必要的，但现在再看这种复杂的认证流程实在觉得没有必要，用起来也麻烦，可能对于过于庞大的权限管理体系来说，复杂本身也是一种防护吧。看看 GitHub 的 OAuth APP 就感觉现代了很多，使用也很方便。

## 登录凭证

除了用户名密码本身，为了增强安全性还需要其他的辅助验证，比如 2FA. 现在很多 App 都支持类似的功能，比如 GitHub/Dropbox/Discord，不过要注意 Auth app 的数据备份，之前我的 Google Authenticator 就陷入了无法找回的情况，痛定思痛之后，便改用了 Authy，这样更方便在新设备上同步。

Credential 的校验一般只是认证的前半部分，通过之后服务器会签发一个凭证给用户，之后一段时间内便不再需要验证了。这么做的主要原因就是为了方便，因为场景中总是存在会话，没人喜欢在过程中每次都要输入密码，类似 sudo. API 调用认证就不会这么做，原因也是没有明显的会话场景。说回凭证，通常是以 Token 的形式出现，HTTP 的认证框架中的 Bearer Auth 就是把 Token 放在 Header 中传递的：`Authorization: Bearer ...`. 当然 Bearer 主要是为了配合 OAuth 2.0，凭证也不一定就是 Token，有两种方式：

1. Cookie: 客户端把 SessionID 放在 Cookie 里面，服务器收到之后检查是否有对应的 Active session. 这样服务端可以完全控制 Session，但你知道，Cookie 的限制是很多的
2. Token: 状态信息都保存在 Token 里，所以服务器是 Stateless 的（至少对于认证来说）。客户端可以把 Token 放在 Header(standardized) 或者 Payload 里发送。好处很明显，服务端减了负，但是控制上就减弱了，比如不能随时 Invalidate 一个 Token; 另外就是 Token 不受局限，可以直接跨域

JWT 是一种很流行的 Token-based 的解决方案，简单来说就是校验通过发给客户端 Token，之后只需要检查 Token 合法性就好了，其中还可以设置 scope 和 expiration. 当然用 JWT 也会遇到控制不足的问题，一种解决办法是 Invalidate refresh token，然后等待 access token 过期（这就要求 expiration 不能太长）。如果有更复杂的需求，可能还是要自定义实现 Token.

# Authorization

以上是认证，再来说授权。其实这两者本来就不分家，只是为了做好权限粒度上的区分，巨头拆出了后者给小公司们用。说实话哪里需要实现自己的用户名密码认证登录，直接提供 Google/FB/Wechat 第三方登录就好了，或者干脆像 TG 那样直接用手机号码。根本没有人想多记一个帐号密码啊。

说回授权，也就是 OAuth 2.0，是一系列精心定义的授权流程，有很多 Flow，但核心其实都在 Authorization Code 这一种里面。比如我做了一个 App，想要支持 GitHub 登录，那么我要先到 GH 注册，得到 ClientID/ClientSecret，可以对照前面提到的 AppID/AppSecretKey. 那么用户在 App 上点击登录，就会先跳到 GH 页面，同意授权之后，跳回 App 同时返回一个 code，那么我的 App 利用这个 code 再结合 ClientSecret 请求 GH 的 access token. 这个过程也不复杂，重点有两个，一是注册得到的密钥对，尤其是 ClientSecret，GH 是通过它来确认 App 的合法身份的；另一个是后端，因为跳回 App 的时候实际是重定向回我预先提供的 `redirect_uri` 的，所以 Authorization Code 适合带有后端服务的应用，保障 Token 的安全性，否则过程中就可能出现漏洞。

Gittalk 是一款纯前端的评论应用，最大的特点是利用 GitHub Issue 来保存评论数据，但因为没有后端，它的授权过程中就并不是很安全。因为没有后端，所以 GH 跳回的 `redirect_uri` 其实是前端页面，在这想要 POST 请求 access token 是跨域禁止的，但 Gittalk 配置了代理服务器来克服这个问题。暂时不管代理服务器是否可靠，ClientID/ClientSecret 是确实保存在前端了的，这就意味着 100% 的泄露可能，另外登录请求的权限也未免太宽了些，看着就不是很愿意授权：

![](https://static.iamgodot.com/content/images/20220128153328.png)

其实 OAuth 2.0 是为纯前端应用提供了 Implicit 这种 Flow 的，就是在回调 `redirect_uri` 的时候直接附上 Token `https://redirect_uri#token=ACCESS_TOKEN`，之所以用锚点也是为了增强安全性。这种方式并不很安全，所以 Token 的过期时间要尽量短。然而 Gittalk 是没机会使用的，因为 GH 明确表示了不支持：

> To authorize your OAuth app, consider which authorization flow best fits your app.
>
> - [web application flow](https://docs.github.com/en/developers/apps/building-oauth-apps/authorizing-oauth-apps#web-application-flow): Used to authorize users for standard OAuth apps that run in the browser. (The [implicit grant type](https://tools.ietf.org/html/rfc6749#section-4.2) is not supported.)
> - [device flow](https://docs.github.com/en/developers/apps/building-oauth-apps/authorizing-oauth-apps#device-flow): Used for headless apps, such as CLI tools.

除了上面两种，还有 Client Credential 这种 Flow. 简单来说，这更类似于认证而不是授权。因为不需要用户首肯，应用直接使用注册得到的 ClientID/ClientSecret 请求 Token，比较适合没有前端的命令行应用。

最后还有一种 Password Flow，用户需要在应用中直接输入 GH 的用户名密码。这其实已经违背了 OAuth 2.0 的初衷，但 FastAPI 利用它来实现应用本身的认证登录，相当于把 Authorization 作为一种 Pseudo Authentication 了。

授权说到这也差不多了。现在的个人/小型应用可能没有必要再实现自己的认证了，登录直接第三方（Apple/Google/GH/WeChat/手机验证码/手机号一键登录），国内外都照顾得到，用户数据依然可以维护，也不用自己管理密码了。至于安全性，HTTP 是一定要加上 S 的。

---

*References*

- [HTTP authentication - MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Authentication)

- [Digest access authentication - Wikipedia](https://en.wikipedia.org/wiki/Digest_access_authentication)

- [Message authentication code - Wikipedia](https://en.wikipedia.org/wiki/Message_authentication_code)

- [HTTP API 认证授权术 - CoolShell](https://coolshell.cn/articles/19395.html)

- [OAuth 2.0 的四种方式 - 阮一峰](https://www.ruanyifeng.com/blog/2019/04/oauth-grant-types.html)

- [Authorizing OAuth Apps - GitHub](https://docs.github.com/en/developers/apps/building-oauth-apps/authorizing-oauth-apps)
