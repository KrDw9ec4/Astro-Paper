---
author: KrDw
pubDatetime: 2024-04-16T22:25:21.000+08:00
modDatetime: 2024-04-20T14:20:27.000+08:00
title: DDNS v Tailscale v  Cloudflared
featured: false
draft: false
tags:
  - homeserver
  - network
description: ""
---

正如[这篇博客](https://blog.krdw.site/posts/building-homeserver-with-laptop/#公网访问)中所说，我外部访问自建服务器用的是 DDNS(v6) + FRP(v4) 以实现 IPv6+IPv4 双栈访问。

然而，不久之后，我发现 SSH 登录出现了问题。经过排查，发现有人通过 FRP 不断扫描端口，导致 SSH 登录受阻。因此，我关闭了 FRP，只用 DDNS 通过 IPv6 进行访问。

因为遇到上面的问题，我总觉得直接暴露端口也不够安全，所以一直在找别的访问方案。搜来搜去，大致也就这三种方案：**DDNS 直连、内网穿透（FRP/CloudFlare Zero Trust）、异地组网（Wireguard/Tailscale）**。

最近几天，把几种方案都浅浅体验了一下，这篇博客就来总结一下感受（**_斜粗体_**=优点，_斜体_=缺点）。

### DDNS

- 因为是 **_IPv6 直连，连接速度只受限于服务器所在网络的上传带宽_**，是连接速度最佳的方案。
- 因为*没有公网 IPv4，客户端仅 v4 环境时无法访问*，但就我目前网络情况（数据流量是 IPv6 优先）而言，绝大多数时间都是 IPv6 访问优先。
- 因为是*直接对外暴露端口，可能会有被爆破的风险*，但不乱传播，仅自用很难会出事。
- ~~如果对外架设 Web 服务，被运营商扫到找上门，还有*可能被掐宽带的风险*。~~（好孩子不要干）

### 内网穿透

#### (1) FRP

- _需要一台有公网 IP 的云服务器_。
- **_配置起来简单直接有效_**，配置好了肯定能用。
- 搭配 DDNS **_可以实现 v6+v4 双栈访问_**，访问地址能做到一致。
- “感觉”_增加了被爆破的风险_，就像我上文所提，开 FRP 的时候有人在扫端口（~~我感觉是阿里云机房在扫~~，因为关了之后再查看相关日志就没有这种记录了。）

#### (2) CloudFlare Tunnel

CloudFlare Tunnel 是 Zero Trust 下属的一个项目，Zero Trust 是可以免费使用的，赛博菩萨（

- _开启 Zero Trust 需要绑定支付方式_。（原本是要境外支付方式的，但境内银联卡 + Paypal 国区也可以。）
- **_对外穿透 HTTP 服务很方便_**，只需要服务端安装 cloudflared 即可，但其他服务比如 _rdp smb ssh 需要客户端也安装 cloudflared_。
- 我认为是一个很完美的方案，相当于在 cloudflare 那边有一个反代服务器，**_直接可以用 https 加相应域名访问，后面不用端口号_**。
- 我认为最大的缺点就是*延迟比较大*，这一点在用 ssh 时尤其明显，键入的命令都有延迟。

### 异地组网

值得一提的是，异地组网和内网穿透使用下来有以下差别：

- **内网穿透只需要在服务端进行相关安装配置，客户端直接就可以访问**；而异地组网需要同时在服务端和客户端安装相关软件。
- 内网穿透仅仅实现了客户端访问服务端；而**异地组网将客户端和服务端纳入一个“大局域网”**，实现电脑手机互访，**防火墙也不用暴露端口**，你甚至可以使用像 LocalSend 这种局域网互传软件，可玩性更高。

#### (1) Tailscale

- 最大的优点就是上手极其简单，服务端和客户端都安装好 Tailscale，**_登录即用_**。
- 同时**_支持 v6 和 v4，IPv6 也能像 DDNS 一样直连_**，IPv4 会走香港中继服务器中转，速度勉强能用。
- 另外，值得一提的是，**_当两台在传统局域网下时，会直接走传统局域网的连接_**。

![](https://img.krdw.me/2024/05/picgo_44003e27bc00a6094f3a1ba615efdab5.png)

![](https://img.krdw.me/2024/05/picgo_51c10841eb748171487e70772112ee4f.png)

- 异地组网的好处就是从局域网访问，**_完全不用对外暴露端口，安全性很高_**。
- 我使用 Tailscale 最大的阻力就是它是一个 VPN 服务，这意味着*在安卓端与科学软件冲突*，而这又是我所必需的。
- 我使用下来还发现一个问题就是如果使用 AuthKey 进行连接，那么每次断开重连都是一台新的设备而不是沿用旧有配置，这也导致访问 IP 地址不固定。

最后经过权衡取舍，我决定还是换用 Tailscale，毕竟我都在 v2 相关帖子下看到好多次 [@Livid](https://www.v2ex.com/member/Livid) 推荐 Tailscale（不是

其实我一开始是打算继续用 DDNS 的，但这篇博客写下来发现 Tailscale 是真好用，只好妥协，手机上通过磁贴快速切换 Tailscale 和科学。

#### (2) Wireguard

- 相比 Tailscale（基于 Wireguard），Wireguard 纯靠配置文件进行连接，**_配置起来麻烦一点，但相应的自由度也更高一点_**。
- 不过每个 peer 都要添加 endpoint，也就是说*需要 ddns 将 peer 的 ip 固定到域名上使用*，这一点还不如 tailscale。
- 貌似科学软件兼容 wireguard 协议，没深入研究。

### 写在最后

我现在自己访问是用 Tailscale + CloudFlare Tunnel，有以下三个好处：

- **安全**：异地组网不用对外暴露端口，避免暴露端口带来潜在被爆破风险。
- **连接性能优异**：Tailscale 本身优异的连接能力，支持 v6、v4、传统局域网多种情况。
- **方便**：使用 CloudFlare Tunnel 作为备用方案用于穿透 HTTP 服务，让客户端能直接访问服务。