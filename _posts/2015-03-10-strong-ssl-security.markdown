---
layout: post
title: 改善 HTTPS 访问安全性
date: 2015-03-10 11:43:23
author: timebug
---

2014 年 4 月 OpenSSL 的心血漏洞 [Heartbleed](http://heartbleed.com/) 着实让大家对互联网安全捏了一把冷汗。同年 10 月份又相继爆出 [POODLE](http://en.wikipedia.org/wiki/POODLE) 安全漏洞攻击，该漏洞可以让攻击者利用 SSLv3 协议设计中的缺陷，通过中间人攻击的手段来窃取用户信息，Google 研究员最先披露了有关该漏洞的[细节](http://googleonlinesecurity.blogspot.co.uk/2014/10/this-poodle-bites-exploiting-ssl-30.html)。

目前 UPYUN 全网已经移除了对 SSLv3 协议的支持，涉及 CDN、API、WEB 等相关 HTTPS 服务。另外，国外知名的 CDN 服务商 [MaxCDN](https://www.maxcdn.com/blog/delivery-sslv3-disabled/) 和 [CloudFlare](https://blog.cloudflare.com/sslv3-support-disabled-by-default-due-to-vulnerability/) 也早在去年 10 月就禁掉了 SSLv3 的支持。

### 移除 SSLv3 有何影响？

所有的现代浏览器和 API 客户端都已经或即将支持 TLSv1.0 及其以上版本的协议，因此该调整对绝大多数的客户端都没有影响，但由于 Windows XP/IE6 或更早的浏览器仅支持 SSLv3，会出现无法访问的问题，据 CloudFlare 的[统计](https://blog.cloudflare.com/sslv3-support-disabled-by-default-due-to-vulnerability/)目前它们全网流量中 3.12% 来自 Windows XP 用户，而其中也只有 1.12% 采用了 SSLv3 的连接方式，换句话说，其他 98.88% 的 Windows XP 用户已经采用了更安全 TLSv1.0+。

以上，考虑到安全原因，对于还在使用 IE6 的用户，我们强烈建议升级你们的浏览器。

### 检查 SSL 服务的安全等级

通过 [QUALYS SSL LABS](https://www.ssllabs.com/ssltest/analyze.html) 提供的检查页面，我们可以详细分析 HTTPS/SSL 服务的安全情况。以 UPYUN 官网 *upyun.com* 为例，我们得到的检查结果如下：

![SSL Report]({{ site.remoteurl }}/assets/sslreport.png)

特别地，低于 A 评级的情况下可能会得到如下一些安全提示：

1. This server supports anonymous (insecure) suites (see below for details). Grade set to F.
2. This server is vulnerable to the POODLE attack. If possible, disable SSL 3 to mitigate. Grade capped to C. [MORE INFO](https://community.qualys.com/blogs/securitylabs/2014/10/15/ssl-3-is-dead-killed-by-the-poodle-attack)
3. This server is vulnerable to the [OpenSSL CCS vulnerability (CVE-2014-0224)](https://community.qualys.com/blogs/securitylabs/2014/06/13/ssl-pulse-49-vulnerable-to-cve-2014-0224-14-exploitable) and exploitable. Grade set to F.
4. Certificate uses a weak signature. When renewing, ensure you upgrade to SHA2. [MORE INFO](https://community.qualys.com/blogs/securitylabs/2014/09/09/sha1-deprecation-what-you-need-to-know)
5. The server supports only older protocols, but not the current best TLS 1.2. Grade capped to B.
6. This server accepts the RC4 cipher, which is weak. Grade capped to B. [MORE INFO](https://community.qualys.com/blogs/securitylabs/2013/03/19/rc4-in-tls-is-broken-now-what)
7. The server does not support Forward Secrecy with the reference browsers. [MORE INFO](https://en.wikipedia.org/wiki/Forward_secrecy)
8. This server does not mitigate the [CRIME](https://community.qualys.com/blogs/securitylabs/2012/09/14/crime-information-leakage-attack-against-ssltls) attack. Grade capped to B.
9. ...

### NGINX SSL 相关优化

由于 UPYUN 线上 HTTPS/SSL 服务都基于 NGINX，因此这里对以上问题的优化也仅针对 NGINX。

> SSL Compression (CRIME attack): [8]

CRIME 攻击主要利用了 SSL Compression 的特性，若以下情况满足就表示该特性已经被默认关闭了：

~~~
* OpenSSL >= 1.0.0: NGINX 1.1.6+/1.0.9+
* OpenSSL <  1.0.0: NGINX 1.3.2+/1.2.2+
~~~

因此，很简单，解决办法就是升级你的 NGINX 和 OpenSSL 即可。

> The POODLE attack (CVE-2014-3566) and TLSv1.0+: [2], [5]

贵宾犬（POODLE）攻击刚上面也详细介绍过了，我们只要在 NGINX 中禁用 SSLv3 协议即可：

~~~
ssl_protocols SSLv3 TLSv1 TLSv1.1 TLSv1.2;
~~~

修改为：

~~~
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
~~~

特别地，TLS 的高版本协议也最好一并加上，目前[部分主流浏览器](http://en.wikipedia.org/wiki/Transport_Layer_Security#Web_browsers)都会优先尝试 TLSv1.2 建立安全连接。

> OpenSSL CCS vulnerability (CVE-2014-0224): [3]

2014 年 6 月 5 号，OpenSSL 团队发布了一份[安全报告](https://www.openssl.org/news/secadv_20140605.txt)，披露了一系列严重漏洞的细节，其中当属 CVE-2014-0224 受影响面最广，主要是由于 OpenSSL 部分版本实现中没有正确处理 ChangeCipherSpec 消息导致的，使得攻击者能够通过中间人攻击来利用这个漏洞，解密并修改被攻击的服务器和客户端之间的通讯信息，从而获得机密数据。

解决办法就是升级 OpenSSL，不同主版本的升级方案如下：

~~~
* OpenSSL 0.9.8 SSL/TLS users should upgrade to 0.9.8za.
* OpenSSL 1.0.0 SSL/TLS users should upgrade to 1.0.0m.
* OpenSSL 1.0.1 SSL/TLS users should upgrade to 1.0.1h.
~~~

> Certificate uses a weak signature: [4]

解决办法：更换证书。证书签名算法需要升级到 SHA2，如果仍然是 SHA1 的话就会得到类似 [4] 的警告，根据 Google 安全团队的[说法](http://googleonlinesecurity.blogspot.jp/2014/09/gradually-sunsetting-sha-1.html)，SHA1 已经被时代抛弃了。

> The Cipher Suite: [1], [6], [7]

经过一些测试，我们最终参考了 CloudFlare 关于 Ciphers 的[配置方案](https://support.cloudflare.com/hc/en-us/articles/200933580-What-cipher-suites-does-CloudFlare-use-for-SSL-)：

~~~
ssl_ciphers EECDH+CHACHA20:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;
~~~

以上配置，在尽可能确保了安全性的前提下，除了上述已知的 Windows XP/IE6 不兼容外，其它绝大多数浏览器和 API 客户端均兼容。另外，对于 [Clpherli.st](https://cipherli.st/) 网站上给出的建议配置（如下），我们在测试过程中发现会额外导致 Windows XP 下 IE7/IE8 出现不兼容的情况，这个基于我们目前的国情是很难接受的：

~~~
ssl_ciphers AES128+EECDH:AES128+EDH;
~~~

### SPDY 以及一些安全相关的 HTTP 响应头

目前 UPYUN 的官网整站都已经开启了 [SPDY](http://en.wikipedia.org/wiki/SPDY) 的支持，通过 [SPDYCheck.org](https://spdycheck.org/#www.upyun.com) 检测即可知晓。

关于 SPDY 协议，最早由 Google 在 Chromium 项目中[提出](http://dev.chromium.org/spdy/spdy-whitepaper)设计草案，SPDY 协议旨在通过压缩、优先级和多路复用降低网页的加载时间和提高数据传输的安全性。目前高版本主流浏览器均已开启对 SPDY 的支持，同时在今年（2015）SPDY 也正式成为了 [HTTP/2](http://en.wikipedia.org/wiki/HTTP/2) 标准协议的基础。

虽然 Google [表示](http://blog.chromium.org/2015/02/hello-http2-goodbye-spdy-http-is_9.html) SPDY 并入 HTTP/2 标准化后考虑在 2016 年移除对 SPDY 的支持，但 HTTP/2 的普及从目前来看，还需要一定时间，NGINX 早些时候也[宣称](http://nginx.com/blog/how-nginx-plans-to-support-http2/)将在 2015 年底完成对 HTTP/2 的支持。因此，在目前的情形下，在 HTTPS 基础上开启 SPDY 支持也绝对是有益无害的。

#### NGINX 开启 SPDY 支持

> 要求 OpenSSL 1.0.1e+, NGINX [1.5.10+](http://nginx.com/news/nginx-inc-announces-second-release-nginx-plus-2/)（支持 SPDY/3.1）。

~~~
./configure --with-http_spdy_module --with-http_ssl_module
~~~

~~~
server {
    listen 443 ssl spdy;
    server_name upyun.com;

    ...
}
~~~

更多详细步骤和配置指令说明请参考 [NGINXTIPS](http://www.nginxtips.com/how-to-install-and-configure-spdy-on-nginx/) 及 [NGINX SPDY](http://nginx.org/en/docs/http/ngx_http_spdy_module.html) 官方文档。

#### NGINX 增加常用安全相关的 HTTP 响应头支持

~~~
add_header Strict-Transport-Security max-age=63072000;
add_header X-Frame-Options SAMEORIGIN;
add_header X-Content-Type-Options nosniff;
add_header X-XSS-Protection "1; mode=block";
~~~

各响应头的作用详见 [List of useful HTTP headers](https://www.owasp.org/index.php/List_of_useful_HTTP_headers)。

### EOF

~~~
server {
    listen 443 ssl spdy;
    server_name upyun.com;

    ssl_certificate       upyun.com.pem;
    ssl_certificate_key   upyun.com.key;

    ssl_ciphers EECDH+CHACHA20:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;

    add_header Strict-Transport-Security max-age=63072000;
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
}
~~~

以上配置在 NGINX 1.7.10 和 OpenSSL 1.0.2 上经过测试，同时也建议大家及时升级到相应的最新版本。
