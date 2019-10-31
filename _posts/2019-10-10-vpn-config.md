---
layout: post
title:  "关于科学上网技术的探讨"
categories: [Dev]
tag: [shadowsocks, vpn]
author: Mikemike Zhu
---

{:toc}

## 前言

最近在学习GFW以及各种科学上网技术的工作原理，查了很多文档和资料，故总结了这篇文章以供大家参考。希望大家在尊重当地法律法规的前提下科学上网。

<br>

## 原理

### 1. GFW的工作原理

GFW (Great Firewall) 主要通过以下几种方法实现：

#### DNS劫持

我们都知道，在网络中，两台机器需要通过IP地址来相互访问。所以当我们访问特定的域名的时候，必须依靠ISP（网络服务提供商）的DNS服务器（Domain Name Service）进行域名解析，从而将域名解析成相应的IP地址。

而GFW则通过DNS劫持（DNS Hijacking）达到阻止访问某些网站的功能。比如说，当我们想要访问google.com的时候，ISP（比如中国移动）就会把这个域名解析到一个错误的IP地址上，从而导致用户无法访问。

![DNS劫持](/assets/img/2019-10-10-vpn-config/gfw_dns.png)

#### IP封锁

GFW还有一种重要的拦截方式是在网络层对IP进行封锁。最早IP封锁是通过ACL访问控制列表（Access Control List）来实现的。当用户访问某个特定服务器的时候，发送的数据请求会经过一系列的路由器进行中转和转发。ACL访问控制列表就如同一个镶嵌在路由器里的一个“黑名单”。当数据请求到达路由器的时候，路由器就会自动将目的IP地址与ACL访问控制列表进行匹配。如果匹配成功，就会执行相应的操作，比如丢弃该数据等。

![ACL访问控制列表](/assets/img/2019-10-10-vpn-config/gfw_acl.png)

然而，ACL访问控制列表也有明显的缺点。例如，当GFW想要拦截的IP地址增多的时候，ACL访问控制列表的匹配操作的时间也会相应地变长。为了解决这个问题，GFW使用路由重分发（Route Redistribution）的功能。首先，GFW的管理员会通过配置静态路由（Static Routing）将发往某个IP的数据请求转发到一个“黑洞服务器”上，这样想要拦截的数据就会被无声无息地丢掉。考虑到静态路由的优先级最高，减少了匹配ACL访问控制列表这一步骤，所以自然效率提高很多。之后，再通过动态路由的路由重分发的功能（Dynamic Routing）将这条错误的路由信息分发到其他骨干路由器上，从而提高了拦截效率。

![路由重分发](/assets/img/2019-10-10-vpn-config/gfw_redistribution.png)

GFW除了会进行IP封锁，也会对具体的端口进行封锁，使得该端口无法接受请求，从而实现更加精确的拦截。

### 2. 常用科学上网方法

#### VPN

VPN（Virtual Private Network）的原本目的并不是科学上网，而是为了让处于公共网络（Public Network）的用户通过某种方式连接到某个私有网络（Private Network），从而实现安全的数据传输。例如，出于安全考虑，公司内网一般无法从外部访问。所以如果公司员工想要在家远程工作，需要使用VPN来接入公司的内网。

VPN的工作原理是，用户的客户端首先与VPN的服务器端经过来回几次认证及密钥，再通过某些协议（如：IPSec等）对数据进行加密。之后VPN的服务器再访问用户需要访问的网站。

由于VPN需要对数据进行多次加密解密，因而影响了数据的传输效率，但确保了传输数据的安全性。

然而，VPN的来回认证以及交换密钥有着明显的流量特征，所以GFW可以通过监测分析流量特征来实现封锁。

#### Shadowsocks

在谈Shadowsocks之前，要首先了解代理服务器（Proxy Server）的概念。

代理服务器的工作原理是，用户的客户端经过代理服务器来访问相关的网站，从而达到突破防火墙，实现科学上网的目的。

而VPN与代理服务器最大的区别在于安全性。VPN服务器会对中间的数据传输进行额外的加密解密，而代理服务器只会隐藏用户的IP地址，一般不涉及额外的加密，数据一般原封不动地到达代理服务器，因此相对缺乏安全性。

代理按照方向主要分为两类：

- 正向代理: 正向代理（Forward Proxy）代替用户访问相关网站，可用于隐藏身份，突破防火墙，以及实现科学上网的目的。

 ![Forward Proxy](/assets/img/2019-10-10-vpn-config/gfw_forward.png)

- 反向代理: 反向代理（Reverse Proxy）一般不用做科学上网，而是用作服务器的负载均衡（Load balancing）。比如说，一般大型网站会有多个服务器，当用户访问的时候，有的服务器处理的请求比较多，有的服务器处理的请求比较少。那么就需要在这些服务器前面加一个反向代理服务器，通过一定的算法把请求平均分配到各个服务器上面。

 ![Reverse Proxy](/assets/img/2019-10-10-vpn-config/gfw_reverse.png)

代理按照代理的协议划分也分为两类：

- HTTP代理: HTTP代理一般用于代理HTTP的请求。比如用户可以通过HTTP代理把HTTP请求转发到目标服务器上。HTTP代理主要工作在应用层（Application Layer）上面，属于比较High-level的一种代理模式。

- SOCKS代理: SOCKS代理跟HTTP代理不同。SOCKS代理工作在会话层（Session Layer），属于比较Low-level的一种代理模式。所以SOCKS代理只是简单传输数据，而不会理会使用何种协议（如HTTP, FTP等）。SOCKS代理也有两种类型，一种是SOCKS4代理，只能代理TCP协议，另一种是SOCKS5代理，不仅可以代理TCP协议，还可以代理UDP协议。

了解完代理服务器，那么我们来说一下Shadowsocks的工作原理。Shadowsocks的数据传输就是建立在SOCKS5上的。主要由两部分构成，一个是建立在本地的ss-local，还有一个建立在代理服务器的ss-server。首先，所有用户的请求会经过ss-local，建立在本地的ss-local收到用户的请求后，通过用户配置的加密方法（例如，AES-256-CFB）进行加密。之后，再转发给建立在防火墙外的ss-server进行处理。ss-server收到请求后，根据用户配置的加密方法对数据进行解密。通过封装的SOCKS5协议，ss-server读出本次请求真正的目标服务地址，再把解密后的原数据转发到真正的目标服务器上。

![Shadowsocks](/assets/img/2019-10-10-vpn-config/gfw_shadowsocks.png)

与VPN不同，Shadowsocks不需要握手（Hand-shaking），虽然解密手段没有VPN安全，但没有明显的流量特征，所以不容易被GFW监测。

<br>

## 参考资料

(1) [Shadowsocks原理详解](http://peachey.blog/2017/12/28/py-shadowsocks/)
<br>
(2) [GFW的拦截原理概析](https://www.svlik.com/1804.html)
<br>
(3) [深入理解GFW：路由扩散技术](http://gfwrev.blogspot.com/2009/11/gfw_05.html)
<br>
(4) [Proxy、VPN、Shadowsocks概念解釋](https://carsonwah.github.io/proxy-vpn-shadowsocks-concept.html)
<br>
(5) [浅谈vpn、vps、Proxy以及shadowsocks之间的联系和区别](https://medium.com/@thomas_summon/%E6%B5%85%E8%B0%88vpn-vps-proxy%E4%BB%A5%E5%8F%8Ashadowsocks%E4%B9%8B%E9%97%B4%E7%9A%84%E8%81%94%E7%B3%BB%E5%92%8C%E5%8C%BA%E5%88%AB-b0198f92db1b)
<br>
(6) [http,socks4,socks5代理的区别](https://my.oschina.net/aiguozhe/blog/127279)
<br>
