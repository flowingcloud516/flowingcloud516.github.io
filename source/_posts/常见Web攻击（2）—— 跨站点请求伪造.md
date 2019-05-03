---
title: 常见Web攻击（2）—— 跨站点请求伪造
date: 2019-05-04 05:41:01
tags:
- http
- web
category: Web前端
---

### 跨站点请求伪造
跨站请求伪造（英语：Cross-site request forgery），也被称为 one-click attack 或者 session riding，通常缩写为 CSRF 或者 XSRF， 是一种挟制用户在当前已登录的Web应用程序上执行非本意的操作的攻击方法。[1] 跟跨网站脚本（XSS）相比，XSS 利用的是用户对指定网站的信任，CSRF 利用的是网站对用户网页浏览器的信任。

### 攻击方式
跨站请求攻击，简单地说，是攻击者通过一些技术手段欺骗用户的浏览器去访问一个自己曾经认证过的网站并运行一些操作（如发邮件，发消息，甚至财产操作如转账和购买商品）。由于浏览器曾经认证过，所以被访问的网站会认为是真正的用户操作而去运行。这利用了web中用户身份验证的一个漏洞：简单的身份验证只能保证请求发自某个用户的浏览器，却不能保证请求本身是用户自愿发出的。

![image](https://note.youdao.com/yws/api/personal/file/WEB511b5c9a997d90485ca38ba071cc0dfb?method=download&shareKey=0d98af5eb3871de1f071696651f3a951 "跨站请求伪造")

1. 用户登录了自己的银行页面 `http://mybank.com`，`http://mybank.com`向用户的cookie中添加用户标识。
1. 用户浏览了恶意页面 `http://evil.com`。执行了页面中的恶意请求代码。
1. `http://evil.com`向`http://mybank.com`发起HTTP请求（还是从浏览器端发送的请求），请求会默认把`http://mybank.com`对应cookie也同时发送过去。
1. 银行页面从发送的cookie中提取用户标识，验证用户无误，response中返回请求数据。此时数据就泄露了。

### 防御措施

#### A. 检查Header中的字段
网站收到带cookie的请求后，为了避免该请求是违背用户意愿的请求，可以检查 Http Header 中的 Referer 或 Origin 
字段，判断请求是否原网站发出。   
由于跨站请求伪造攻击，必须让用户从浏览器发送请求，才能获取对应cookie，达到攻击的目的。而从浏览器发送的请求不受 Evil 网站控制，也不好伪造 Header 
信息。所以一般情况下可以防止用户被跨站请求伪造攻击。   
**优点**：简单易行，工作量低，仅需要在关键访问处增加一步校验。   
**缺点**：验证Header的措施强依赖浏览器的可靠性。如果浏览器也被攻击，则无法起到正常的防御作用。

#### B. 增加 token 验证
简单来说，除了cookie识别访问者身份外，再加一层保护。
1. **Synchronizer token pattern**：令牌同步模式（Synchronizer token pattern，简称STP）是在用户请求的页面中的所有表单中嵌入一个token，在服务端验证这个token的技术。token可以是任意的内容，但是一定要保证无法被攻击者猜测到或者查询到。攻击者在请求中无法使用正确的token，因此可以判断出未授权的请求。
1. **Cookie-to-Header Token**：对于使用Js作为主要交互技术的网站，将CSRF的token写入到cookie中，然后使用javascript读取token的值，在发送http
请求的时候将其作为请求的header（用攻击者猜测不到的Header项），最后服务器验证请求头中的token是否合法。   
Token 验证基于一种假设，即攻击者猜不出token的正确使用方式，来完成鉴权。

#### C. Same-Site Cookies
参考[2]中提到的这种 Cookie 实际上是给 cookie 带上一种同站访问的属性，比如：
```
Set-Cookie: sess=abc123; path=/; SameSite
```
它告诉浏览器将采用何种方式来保护 cookie。
1. Strict 模式
在 strict 模式下，任何**跨域**请求都不会携带cookie进行操作，从根本上杜绝了CSRF的发生。   
但是这种模式下，顶级导航（直接在地址栏改变 URL ）的请求都不会携带 cookie。比如说有一个链接地址 facebook
.com 并且 Facebook 的 SameSite cookies 的模式为 Strict，当你点击链接打开 Facebook 之后你会发现你无法登录。无论你之前是否登录。
2. Lax 模式
将 SameSite 保护设置为 Lax 模式将会解决上面提到的在 Strict 模式下的用户在已经登录的前提下点击链接仍然无法在目标网站登录的问题。在 Lax 模式下有一个例外，就是在顶级导航中使用一个安全的 HTTP 方法发送的请求可以携带 cookie。所谓 "安全的" 的 HTTP 方法在 Section 4.2.1 of RFC 7321 定义为 GET、HEAD、OPTIONS 和 TRACE。   
但是在 Lax 模式下还有一些需要注意的点。比如，如果一个攻击者触发一个顶级导航或者弹出一个新的窗口，那么他们就可以让浏览器发送一个携带 cookies 的 GET 请求。
3. 弊端
该手段同样需要浏览器支持，在浏览器被挟持的条件下无法提供有效的保护。   
另外，某些主流浏览器的低版本不支持这一特性。

#### 其他验证
比如可以使用验证码可以杜绝CSRF攻击，但是这种方式要求每个请求都输入一个验证码，显然没有哪个网站愿意使用这种粗暴的方式，用户体验太差。

### 跨域资源共享 CORS (Cross-origin resource sharing)
总有那么一些场景，需要XHR跨域进行访问。CORS由此而生。
#### 基本原理
浏览器将CORS请求分为简单请求（simple request）和非简单请求（not-so-simple request）   
满足以下2个条件，即为简单请求：
1. 请求方法是 [HEAD, GET, POST] 之一
1. HTTP 头信息不超过这几种字段：
    1. Accept
    2. Accept-Language
    3. Content-Language
    4. Last-Event-ID
    5. Content-Type：只限于三个值application/x-www-form-urlencoded、multipart/form-data、text/plain
    
对于简单请求，CORS会在请求头**加一个字段**；而对于非简单请求，CORS会通过**预请求**的方式进行解决。
    
#### 简单请求跨域
对于简单请求，浏览器会在请求头信息中，加入一个`Origin`字段。该字段说明本请求来自于哪个源。服务器将根据这个值，判断是否同意此次请求。
```
GET /cors HTTP/1.1
Origin: http://api.bob.com
Host: api.alice.com
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0...
```
如果服务器拒绝，则会返回一个错误。   
如果服务器同意请求，Response的头信息中会多加入几个字段
```
Access-Control-Allow-Origin: http://api.bob.com # 必须字段。 要么是请求时Origin的值，要么是一个 *
Access-Control-Allow-Credentials: true # 可选字段。表示是否允许发送Cookie
Access-Control-Expose-Headers: FooBar  # 可选字段。CORS请求时，XMLHttpRequest对象的getResponseHeader()方法只能拿到6个基本字段：:
                                       # Cache-Control、Content-Language、Content-Type、
                                       # Expires、Last-Modified、Pragma
                                       # 如果想拿到其他字段，就必须在Access-Control-Expose-Headers里面指定。上面的例子指定，getResponseHeader('FooBar')可以返回FooBar字段的值。
Content-Type: text/html; charset=utf-8
```
### withCredentials 属性
如果要把Cookie发送到服务器，一方面要服务器同意，另一方面，请求必须打开 withCredentials 属性。如
```javascript
var xhr = new XMLHttpRequest();
xhr.open('GET', 'http://example.com/', true);
xhr.withCredentials = true;
xhr.send(null);
```
需要注意，如果要发送cookie，`Access-Control-Allow-Origin`就不能设为星号，必须指定明确的Origin。

#### 非简单请求跨域
非简单请求会在正式请求之前，增加一次http请求，以保证跨域安全。
1. **预请求**
此次请求浏览器先询问服务器，当前网页所在的域名是否在服务器的许可名单之中，以及可以使用哪些HTTP动词和头信息字段。只有得到肯定答复，浏览器才会发出正式`的XMLHttpRequest`请求，否则就报错。   
预请求头示例：
```javascript
OPTIONS /cors HTTP/1.1
Origin: http://api.bob.com
Access-Control-Request-Method: PUT  # 列出CORS请求会用到哪些方法
Access-Control-Request-Headers: X-Custom-Header #  指定CORS会额外发送的header字段
Host: api.alice.com
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0...
```
预请求方法是`OPTION`，关键字段 `Origin`用于表明身份。详见参考 [3]。

2. **预请求回应**
服务器检查了 `Origin`、`Access-Control-Request-Method`和`Access-Control-Request-Headers`字段以后，确认允许跨源请求，就可以做出回应。
    1. 如果服务器同意请求，会在response的header中加入 `Access-Control-Allow-Origin` 字段，该字段值可以是 * ，表示同意此次跨域
    2. 如果服务器拒绝请求，会返回一个正常HTTP回应，但是不带 `Access-Control-Allow-Origin` 字段。此时，浏览器会认为服务器拒绝了跨域请求，并进行错误流程。

### _References_:
- [1]: [防范CSRF跨站请求伪造](https://segmentfault.com/a/1190000008505616)
- [2]: [跨站请求伪造已死（译）](https://juejin.im/post/58c669b6a22b9d0058b3c630)
- [3]: [阮一峰-CORS跨域](http://www.ruanyifeng.com/blog/2016/04/cors.html)