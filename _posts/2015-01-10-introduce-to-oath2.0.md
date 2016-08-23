---
layout: post
title: 开放授权协议OAuth2.0简介
categories: [Android]
tags: [Optimization]
---

可能你跟我一样，使用过各种第三方开放授权库(如在你的 APP 中获取 QQ 照片或微博评论等)来获取用户的一些资源，今天跟大家总结分享一下开放授权(OAuth2.0，1.0太复杂已经被弃用)的概念和原理，在以后使用开放授权SDK时能快速高效完成。

![01](/album/2015/2015-01-09-introduce-to-oath2-1.png)

##OAuth解决了什么问题？

OAuth 产生主要解决了第三方应用访问用户在某网站网络资源的问题。例如，上图中，用户在36氪登陆时，可以用 QQ 来登陆，点击用 QQ 登陆后，就出现了第三方应用36氪想获取用户基本的 QQ 信息已经微博评论，然后快速登陆的过程；再例如某个图片网站想获取你 QQ 空间的照片，那么就需要跟 QQ 申请获取空间图片的权限，而 QQ 会返回界面，让用户来选择是否授权。


## OAuth协议的角色

- Resource Owner(RO):资源所有者，在上面的例子中是 QQ 用户。
- Resource Server(RS):资源服务器，在上面例子中是 QQ 微博评论和空间照片这两个资源 Server，存储并处理对这些资源的访问。
- Authorization Server(AS):授权服务器，认证资源所有者身份并



根据 OAuth2.0 的 [RFC 6749](https://tools.ietf.org/html/rfc6749)，其流程如下图所示：

![02](/album/2015/2015-01-09-introduce-to-oath2-2.png)


**A.** 用户使用某应用后，此应用要求用户授权访问一些资源。例如第一幅图中右下角所示 “36氪将获取以下权限...”。

**B.** 用户同意给此应用授权(Grant)。例如第一幅图，用户点击 QQ 账号确认。

**C.** 应用使用 B 步骤用户的授权，向认证服务器(AS)申请*访问令牌(Access Token)*。

**D.** 认证服务器(AS)验证通过后，向应用返回*访问令牌(Access Token)*。

**E.**  应用使用获取到的*访问令牌*，向资源服务器(RS)获取资源。

**F.** 资源服务器验证访问令牌的有效性，验证通过，给应用提供相应的资源。


OAuth2.0 提出多种授权类型来支持不同类型的第三方应用，有**授权码(Authorization Code Grant)、隐式授权 (Implicit Grant)、RO凭证授权 (Resource Owner Password Credentials Grant)、Client凭证授权 (Client Credentials Grant),** 我们只看最广泛使用的、功能最完整严密的**授权码**模式。其他可以参考 [RFC 6749](https://tools.ietf.org/html/rfc6749)。



## 授权码模式

如下图所示，授权码模式如下：

![02](/album/2015/2015-01-09-introduce-to-oath2-3.png)


**A.** 用户访问应用，应用将用户引向认证服务器。如第一幅图，36氪想获取QQ用户一些资源，首先将用户引向QQ的授权页面(授权服务器返回)。

应用申请认证的 URL，可包含以下参数：

- response_type：这个值必须是 "code"，用于获取授权码(authorization code)；必选。
- client_id：应用的 ID 名称；必选。
- redirect_uri：重定向链接；可选。
- scope：应用申请权限的范围；可选。
- state：当前应用状态，应用可指定任意值，AS返回此值，主要用于抵制 CSRF 攻击；可选。

**B.** 用户在授权服务器返回的页面上选择是否给予此应用授权（用户必须显式同意或拒绝）。如第一幅图，QQ授权服务器返回一个界面，有36氪想获取的资源，用户来决定是否给予这些授权。

**C.** 如果用户同意授权，认证服务器将用户转向应用已经指定的 *重定向链接(redirection URI)*，重定向链接包必须含一个授权码(authorization_code)；不同意则通过 *重定向链接(redirection URI)* 返回对应的 ERROR 信息。

认证服务器返回给应用的 URI，需要包含的参数如下：

- code： AS产生的授权码，为了安全起见，授权码的有效期一般为10分钟，应用拿到后只能使用一次；多余一次 AS 拒绝服务。授权码与应用 ID 以及重定向 URL 一一对应；必选。
- state： 若应用的请求中包含这个参数，认证服务器的回应也必须一模一样包含这个参数，主要为了抵制 CSRF 攻击；必选。

认证服务器返回 ERROR 信息，主要包含：

- error: 一个 ASCII 编码的 error code，其值意义包括 invalid_request、unauthorized_client、access_denied、unsupported_response_type、invalid_scope、server_error、temporarily_unavailable；必选。
- error_description：对错误的描述；可选。
- error_uri：对于错误的附加信息，比如出现了这个错误，然后指向错误的帮助页面；可选。
- state：state值如果应用请求的信息里有，必须给予相同值的回复；必选。

**D.** 应用收到授权码，加上之前的重定向链接和认证应用身份的数据，访问认证服务器(AS)来获取授权令牌(Access Token)。

应用向认证服务器申请访问令牌的HTTP请求，包含以下参数

- grant_type：值必须是 "authorization_code";必选。
- code：认证服务器返回的授权码；必选。
- redirect_uri：表示重定向 URI，且必须与A步骤中的该参数值保持一致；必选。
- client_id：表示应用ID；必选项。

**E.** 认证服务器收到 D 中的数据后，首先验证应用合法性，其次验证 redirect_uri 与 C 中的是否一致，验证通过后，返回*授权令牌*。

认证服务器发送的HTTP回复，包含以下参数：

- access_token：表示访问令牌；必选项。

- token_type：令牌类型，该值大小写不敏感，可以是 bearer 类型或 mac 类型；必选项。

- expires_in：过期时间，单位为秒。如果省略该参数，必须其他方式设置过期时间；推荐使用。

- refresh_token：更新令牌，用来获取下一次的访问令牌，可选项。

- scope：权限范围，如果与客户端申请的范围一致，可省略此项。

## 举例说明授权码模式

这个例子是[The OAuth 2.0 Authorization Framework-RFC6749](https://tools.ietf.org/html/rfc6749)上的。

与授权码模式的 A B C D E 分别对应。

**A**

> GET /authorize?response_type=code&client_id=s6BhdRkqt3&state=xyz&redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb HTTP/1.1
> Host: server.example.com

**B**

> 用户选择同意还是拒绝，这里没有产生动作。

**C**

> HTTP/1.1 302 Found
> Location: https://client.example.com/cb?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz

ERROR 时返回：

> HTTP/1.1 302 Found
> Location: https://client.example.com/cb#error=access_denied&state=xyz

**D**

> POST /token HTTP/1.1
>     Host: server.example.com
>     Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
>     Content-Type: application/x-www-form-urlencoded

>     grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA&redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb

**E**

>   HTTP/1.1 200 OK
>     Content-Type: application/json;charset=UTF-8
>     Cache-Control: no-store
>     Pragma: no-cache

>     {
>       "access_token":"2YotnFZFEjr1zCsicMWpAA",
>       "token_type":"example",
>       "expires_in":3600,
>       "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
>       "example_parameter":"example_value"
>     }


还可以参考第三方开放授权的文档，比如 [QQ}(http://wiki.open.qq.com/wiki/mobile/OAuth2.0%E7%AE%80%E4%BB%8B)，[新浪微博](http://open.weibo.com/wiki/Oauth)等，都有对他们自己开放授权 SDK 的详细介绍。