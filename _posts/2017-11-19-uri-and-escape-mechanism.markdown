---
layout: post
title: "浅谈 URI 及其转义"
data: 2017-11-19 17:51:34
author: tokers
comments: true
---

URI
----

URI，全称是 Uniform Resource Identifiers，即统一资源标识符，用于在互联网上标识一个资源，比如 `https://www.upyun.com/products/cdn` 这个 URI，指向的是一张漂亮的，描述又拍云 CDN 产品特性的网页。

URI 的组成
---------

完整的 URI，由四个主要的部分构成：

`    <scheme>://<authority><path>?<query>`

`scheme` 表示协议，比如 `http`，`ftp` 等等，详细介绍可以参考 [rfc2396#section-3.1](https://tools.ietf.org/html/rfc2396#section-3.1)。

`authority`，用 `://` 来和 `scheme` 区分。从字面意思看就是“认证”，“鉴权”的意思，引用 [rfc2396#secion-3.2](https://tools.ietf.org/html/rfc2396#section-3.2) 的一句话：
> This authority component is
   typically defined by an Internet-based server or a scheme-specific
   registry of naming authorities.

这个“认证”部分，由一个基于 Internet 的服务器定义或者由命名机关注册登记（和具体的协议有关）。

而常见的 `authority` 则是：“由基于 Internet 的服务器定义”，其格式如下：

`<userinfo>@<host>:<port>`

`userinfo` 这个域用于填写一些用户相关的信息，比如可能会填写 "user:password"，当然这是不被建议的。抛开这个不讲，后面的 `<host>:<port>` 则是被熟知的服务器地址了，`host` 可以是域名，也可以是对应的 IP 地址，`port` 表示端口，这是一个可选项，如果不填写，会使用默认端口（也是和协议相关，比如 `http` 协议默认端口是 80）。

`path`，在 `scheme` 和 `authority` 确定下来的情况下标识资源，`path` 由几个段组成，每个段用 `/` 来分隔。注意，`path` 不等同于文件系统定义的路径。

`query`，查询串（或者说参数串），用 `?` 和 `path` 区分开来，其具体的含义由这个具体资源来定义。


保留字符
-------

从上面的描述里看，URI 的这 4 个组件，由特定的分隔符来分离，这些分隔符各自有着特殊含义，而如果这些分隔符出现在某个组件内，比如 `path` 是 `/a/b?c.html`，那么从 URI 整体角度来看的话， `c.html` 会被当做是 `query`，这样就破坏了 `path` 原本的含义，因此 URI 引入了保留字符集，这些字符有着特殊的目的，如果它们被用于描述资源（而不是作为分隔符出现），那么必须对它们转义。

那么什么情况下需要对一个字符转义呢，引用 [rfc2395#section-2.2](https://tools.ietf.org/html/rfc2396#section-2.2) 的一句话：
> In general, a character is
> reserved if the semantics of the URI changes if the character is
> replaced with its escaped US-ASCII encoding.

即如果转义前后这个字符会影响到整个 URI 的意义，则它必须被转义。

由于 URI 由多个组件构成，一个字符不转义，可能会对其中一个组件会造成影响，但对另一个组件没有影响，所以“保留字符集”是由具体的 URI 组件来规定的。
                    
* 对 `path` 部分而言，保留字符集是（参考自 rfc2396）：

`reserved = "/" | "?" | ";" | "="`

* 对 `query` 部分而言，保留字符集是（参考自 rfc2396）：

`reserved = ";" | "/" | "?" | ":" | "@" | "&" | "=" | "+" | "," | "$"`


字符的转义规则如下：

```
escaped     = "%" hex hex
hex         = digit | "A" | "B" | "C" | "D" | "E" | "F" |
                      "a" | "b" | "c" | "d" | "e" | "f"
```

比如 `,` 转义后为 `%2C`。


特殊字符
-------

有一类不被允许用在 URI 里的特殊字符，它们被称为控制字符，即 ASCII 范围在0-31 之间的字符，以及 ASCII 码为 127 的这个字符。比如 `\t`，`\a` 这些（不包括空格），因为这些字符不可打印而且在某些场景下可能会消失。

另外一类则是扩展 ASCII 码，即范围 128-255 的那些字符，它们不属于 "US-ASCII coded character set"，因此这些字符如果出现在 URI 中，需要被转义。

> URLs are written only with the graphic printable characters of the
   US-ASCII coded character set. The octets 80-FF hexadecimal are not
   used in US-ASCII, and the octets 00-1F and 7F hexadecimal represent
   control characters; these must be encoded.


不安全字符
---------

> Characters can be unsafe for a number of reasons.  The space
   character is unsafe because significant spaces may disappear and
   insignificant spaces may be introduced when URLs are transcribed or
   typeset or subjected to the treatment of word-processing programs.
   The characters "<" and ">" are unsafe because they are used as the
   delimiters around URLs in free text; the quote mark (""") is used to
   delimit URLs in some systems.  The character "#" is unsafe and should
   always be encoded because it is used in World Wide Web and in other
   systems to delimit a URL from a fragment/anchor identifier that might
   follow it.  The character "%" is unsafe because it is used for
   encodings of other characters.  Other characters are unsafe because
   gateways and other transport agents are known to sometimes modify
   such characters. These characters are "{", "}", "|", "\", "^", "~",
   "[", "]", and "`".
   
 这段话引用自 [rfc1738 2.2 节](https://www.ietf.org/rfc/rfc1738.txt)。因为种种的原因，存在一类字符，它们是 "unsafe" 的，不加处理地存在在 URI 里，会破坏 URI 的语义完整性，对于这类字符，如果要出现在 URI 里，那么也得进行转义。

nginx 的 URI 转义机制
-------------------

nginx （以现在最新的 1.13.8 版本为准）提供了一个名为 `ngx_escape_uri` 的函数，函数原型如下：

```c
uintptr_t ngx_escape_uri(u_char *dst, u_char *src, size_t size,
 	ngx_uint_t type);
```

第三个参数，`type`，可以接受这些值：

```c
#define NGX_ESCAPE_URI            0
#define NGX_ESCAPE_ARGS           1
#define NGX_ESCAPE_URI_COMPONENT  2
#define NGX_ESCAPE_HTML           3
#define NGX_ESCAPE_REFRESH        4
#define NGX_ESCAPE_MEMCACHED      5
#define NGX_ESCAPE_MAIL_AUTH      6
```

我们只关心其中的 `NGX_ESCAPE_URI `，`NGX_ESCAPE_ARGS `，`NGX_ESCAPE_URI_COMPONENT `，根据 nginx 官方所提供的 [nginx 模块和核心 API 介绍](https://www.nginx.com/resources/wiki/extending/api/)，这三个宏的含义如下：

| Type | Definition |
|-----|---------|
|NGX_ESCAPE_URI	|  Escape a standard URI |
|NGX_ESCAPE_ARGS	| Escape query arguments|
|NGX_ESCAPE_URI_COMPONENT|	Escape the URI after the domain|

对应地，`ngx_escape_uri ` 这个函数，内置了几个相关的 `bitmap`，区别就是在于各自的转义字符集，具体可以查阅 nginx 的源码（src/core/ngx_string.c）。

其中针对整个 URI 的转义处理，`ngx_escape_uri ` 会把 `" ", "#", "%", "?"` 以及 `%00-%1F` 和 `%7F-%FF` 的字符转义；针对 `query` 的转义，会把 `" ", "#", "%", "&", "+", "?"` 以及 `%00-%1F` 和 `%7F-%FF` 的字符转义；针对 `path` + `query`（称之为 the URI after the domain）的转义，会把除英文字母，数字，以及 `"-", ".", "_", "~"` 这些以外的字符全部转义。

可以看到，`NGX_ESCAPE_URI ` 和 `NGX_ESCAPE_ARGS ` 没有处理不安全字符，前者站在处理整个的 URI 的角度上编码，后者站在处理 `query` 的角度上编码；而 `NGX_ESCAPE_URI_COMPONENT `，处理角度不是整个 URI，而是 domain 之后的 URI 组件，它兼顾 `path` 和 `query` 的保留字符集，更加严格，遵守了 [rfc3986#section-2.2](#https://tools.ietf.org/html/rfc3986#section-2.2) 的规范。

这里顺便提一下 `ngx_proxy` 模块对应的 URI 转义处理，在构造向上游发送的请求行时，ngx_proxy 模块针对 `proxy_pass` 指令做出了不同的处理：

* 如果指定的 URI 包含了变量，将解析变量，然后直接将解析后的 URI 发送到上游；
* 如果 URI 不含变量，且没有指定 `path` 部分，将使用客户端发来的 `path` 部分拼接到 URI 中，然后发送到上游；
* 如果URI 不含变量，且指定了 `path`，这里的处理比较特殊，nginx 会把解码过的，由客户端发来的 URI 里的 `path` 部分（去掉和当前 `location` 的公共前缀），进行编码（按 `NGX_ESCAPE_URI` 来操作），和 `proxy_pass` 指令指定 的 `path` 拼接，发送到上游，比如这样的配置：

```nginx

location /foo {
    proxy_pass http://127.0.0.1:8082/bar;
}
```

如果客户端发来的 URI 里 `path` 是 `/foo/%5B-%5D`，最终上游的 URI `path` 会是 `/bar/[-]`。

因此我们在做 nginx conf 配置的时候，也需要小心考虑 URI 编码的问题。

ngx_lua 的 URI 转义机制
-----------------------

ngx_lua 提供的 `ngx.escape_uri` 函数，和 nginx 核心的转义机制也有一些差异（基于 ngx_lua v0.10.11），体现在对保留字符的处理上，`ngx.escape_uri ` 底层使用的 `ngx_http_lua_escape_uri`，结构和 `ngx_escape_uri` 一致，而对应的 `bitmap` 不同。

对于整个 URI 的转义处理，在 `ngx_escape_uri` 的基础上，对 `'"', '&', '+', '/', ':', ';', '<', '=', '>',  '[', '\', ']', '^', '_', '{' , '}'`进行转义；对于 `query` 的处理，这里去掉了 `&` 的转义；对于 `path` + `query` 的处理，去掉了对 `"'", "*", ")", "(", "!"` 的转义。目前 `ngx.escape_uri` 使用的是 `NGX_ESCAPE_URI_COMPONENT `，从 PR 提交的信息来看，目前 `ngx.escape_uri` 的行为和 Chrome JS 实现的 `encodeURIComponent` 一致。

另外，ngx_lua 对 URI 的解码操作，除了它把 `+` 解码为空格以外，其他和 nginx 相同。

总结
----

在做相关的代理服务，网关服务时，URI 的编解码处理都是非常重要的，某些场景我们可能需要用 URI 来做 key（比如作为 hash 函数的因子），如果不处理好编解码问题，可能在 URI 复杂的情况下会达不到我们的预期效果，反而会浪费很多时间去排查问题的原因，特别地，在使用  nginx 和 ngx_lua 做服务时，我们更应该熟知它们在 URI 编解码上的区别，在理解它们的区别上做自身的业务处理，避免踩坑。

参考资料
-------

* https://www.ietf.org/rfc/rfc1738.txt
* https://tools.ietf.org/html/rfc2396
* https://tools.ietf.org/html/rfc3986
* https://www.nginx.com/resources/wiki/extending/api/utility/#ngx-escape-uri
* https://en.wikipedia.org/wiki/Uniform_Resource_Identifier
