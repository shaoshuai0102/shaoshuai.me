---
layout: post
title: "Cookie Theft and Session Hijacking"
date: 2014-08-07 23:05
comments: true
categories: tech
---

## cookie的基本特性

如果不了解cookie，可以先到[wikipedia](http://en.wikipedia.org/wiki/HTTP_cookie)上学习一下。

### http request

浏览器向服务器发起的每个请求都会带上cookie：

    GET /index.html HTTP/1.1
    Host: www.example.org
    Cookie: foo=value1;bar=value2
    Accept: */*

### http response

服务器给浏览器的返回可以设置cookie：

    HTTP/1.1 200 OK
    Content-type: text/html
    Set-Cookie: name=value
    Set-Cookie: name2=value2; Expires=Wed,09 June 2021 10:18:32 GMT

    (content of page)


## cookie有关的术语

### session cookie

当cookie没有设置超时时间，那么cookie会在浏览器退出时销毁，这种cookie是session cookie。

### persistent cookie/tracking cookie

设置了超时时间的cookie，会在指定时间销毁，cookie的维持时间可以持续到浏览器退出之后，这种cookie被持久化在浏览器中。

很多站点用cookie跟踪用户的历史记录，例如广告类站点会使用cookie记录浏览过哪些内容，搜索引擎会使用cookie记录历史搜索记录，这时也可以称作tracking cookie，因为它被用于追踪用户行为。

### secure cookie

服务器端设置cookie的时候，可以指定`secure`属性，这时cookie只有通过https协议传输的时候才会带到网络请求中，不加密的http请求不会带有secure cookie。

设置secure cookie的方式举例：

    Set-Cookie: foo=bar; Path=/; Secure

### HttpOnly cookie

服务器端设置cookie的时候，也可以指定一个`HttpOnly`属性。

    Set-Cookie: foo=bar; Path=/; HttpOnly

设置了这个属性的cookie在javascript中无法获取到，只会在网络传输过程中带到服务器。

###third-party cookie

第三方cookie的使用场景通常是iframe，例如www.a.com潜入了一个www.ad.com的广告iframe，那么www.ad.com设置的cookie属于不属于www.a.com，被称作第三方cookie。

### supercookie

cookie会从属于一个域名，例如www.a.com，或者属于一个子域，例如b.a.com。但是如果cookie被声明为属于.com会发生什么？这个cookie会在任何.com域名生效。这有很大的安全性问题。这种cookie被称作supercookie。

浏览器做出了限制，不允许设置顶级域名cookie(例如.com，.net)和pubic suffix cookie(例如.co.uk，.com.cn)。

现代主流浏览器都很好的处理了supercookie问题，但是如果有些第三方浏览器使用的顶级域名和public suffix列表有问题，那么就可以针对supercookie进行攻击啦。

### zombie cookie/evercookie

僵尸cookie是指当用户通过浏览器的设置清除cookie后可以自动重新创建的cookie。原理是通过使用多重技术记录同样的内容(例如flash，silverlight)，当cookie被删除时，从其他存储中恢复。

evercookie是实现僵尸cookie的主要技术手段。

了解[僵尸cookie](http://en.wikipedia.org/wiki/Zombie_cookie)和[evercookie](http://en.wikipedia.org/wiki/Evercookie)。

## cookie有什么用

通常cookie有三种主要的用途。

### session管理

http协议本身是是无状态的，但是现代站点很多都需要维持登录态，也就是维持会话。最基本的维持会话的方式是[Base Auth](http://en.wikipedia.org/wiki/Basic_access_authentication)，但是这种方式，用户名和密码在每次请求中都会以明文的方式发送到客户端，很容易受到中间人攻击，存在很大的安全隐患。

所以现在大多数站点采用基于cookie的session管理方式：

用户登陆成功后，设置一个唯一的cookie标识本次会话，基于这个标识进行用户授权。只要请求中带有这个标识，都认为是登录态。

### 个性化

cookie可以被用于记录一些信息，以便于在后续用户浏览页面时展示相关内容。典型的例子是购物站点的购物车功能。

以前Google退出的iGoogle产品也是一个典型的例子，用户可以拥有自己的Google自定制主页，其中就使用了cookie。

### user tracking

cookie也可以用于追踪用户行为，例如是否访问过本站点，有过哪些操作等。

## cookie窃取和session劫持

本文就cookie的三种用途中session管理的安全问题进行展开。

既然cookie用于维持会话，如果这个cookie被攻击者窃取会发生什么？session被劫持！







 
