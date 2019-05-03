---
title: 常见Web攻击（1）—— 代码注入攻击
date: 2019-04-20 22:07:22
tags: 
- http
- web
category: Web前端
---


从安全的角度来看，Http协议具有以下特点：
- 单纯Http在协议层面不具备必要的安全功能
- 在客户端可篡改Http请求，使得攻击更容易实现   

因此，在对外提供Http服务时，要保持不信任用户输入的原则，谨慎防范可能发生的Http攻击。
在此小结一下比较常见的代码注入类Web攻击，以及对应的防范方法。

### 所谓代码注入
众所周知，程序是由代码编译后，交给计算机执行。如果能 1)直接修改代码，将攻击逻辑注入，而 2)计算机不去验证代码的权限，依旧会执行此类具有攻击逻辑的代码。由此，攻击行为完成。此类攻击主要有 SQL/OS脚本 
注入、跨站脚本攻击（XSS攻击）、HTTP首部注入
### 1. SQL 注入
SQL注入，指将前端通过表单或输入提交，将部分SQL命令注入到后端执行SQL中，达到修改后端逻辑目的的一种攻击方式。通常发生在后端SQL拼接时未对前端传入参数进行校验过滤的场景。
##### SQL 注入原理   
如后端有SQL如下，需要前端传入一个 `id` 值，以进行信息查询：
```sql
SELECT * FROM `MyTable` WHERE `id`="{0}"
```
而此时，如果前端传入 `1" OR TRUE`。后端没有校验便传入参数值并对SQL进行拼接。那么直接拼接的SQL为：
```sql
SELECT * FROM `MyTable` WHERE `id`="1" OR TRUE
```
此时将所有数据全部取出。如果后端SQL是`DELETE`，影响更是不可估量。
##### SQL 注入的防范   
防范思路主要有2种：**1. 参数校验（前端或后端进行）**；**2.SQL预编译，后传参**。     
首先我们要确立一个原则，即`不信任任何用户输入信息`。尤其是用户输入的 Plain Text。因为我们不知道用户是什么水平，他们输入的信息对我们的系统是否会造成巨大伤害。   
在此理念的指导下，对于用户提交的输入信息，我们需要进行充分的校验，避免其对后端服务系统造成攻击。相关手段可以是 正则式检验参数合法性、html元素校验、SQL元素校验等，并且可以在前端/后端执行。   
如果校验逻辑在前端执行，确实不会对后端性能造成影响，但对于有经验的攻击者，可以修改前端js代码，此屏障就形同虚设；而如果在后端进行校验，多少会损耗后端性能，如果校验算法比较复杂耗时，也许又会成为攻击者的另一个攻击点。
##### SQL 预编译
MySQL 提供 `prepare`语句，使用户可以传入一个SQL模板，而后续传入参数作为纯粹的String值进行处理。另外，MyBatis 中，如果 SQL 如下书写，也可以使用预编译的方式传递SQL：
```sql
SELECT * FROM MyTable WHERE `id`=#{id}
```
此文不对此进行展开。详情可参考 \[5\] \[6\] 。

### 2. OS 命令注入
该类和SQL代码注入类似，不同在于拼接的命令是SQL还是Bash。防御方案可采用校验法，即在命令执行之前，校验用户输入是否合法。   
针对此类攻击，目前笔者没有找到类似SQL预编译的解决方案。由此可见，根据用户输入信息，来拼接具体执行功能的代码，是多么不明智的举动。

### 3. XSS 跨站脚本攻击
全称跨站脚本攻击（Cross-Site Scripting, XSS），是指通过存在安全漏洞的Web网站注册用户的浏览器内，运行非法HTML标签或JS代码的一种攻击方式。   
定义是从\[4\]抄下来的，看着很眼晕啊，我们用两个栗子来解释一下。   

##### 输入HTML/JS代码进行攻击
在用户输入时，如果前端不进行校验，用户可能输入一些非法html、甚至是 JS 代码，来达到攻击的目的。如
![image](/images/blog/web/xss-dynamic_webpage.png "用户输入 < s > 标签，更改页面逻辑（图来自参考资料4，侵删）")
又或者，页面
```html
<input type="text" value="<%= getParameter("keyword") %>">
<button>搜索</button>
<div>
  您搜索的关键词是：<%= getParameter("keyword") %>
</div>
```
而用户输入为 `"><script>alert('XSS');</script>`。经过拼接，html 将执行 `alert('XSS')`。
XSS 攻击详见参考资料\[2\]。

### 4. Http 首部注入
HTTP 首部注入攻击(HTTP Header Injection)是指攻击者通过在响应首部字段内插入换行,添加任意响应首部或主体的一种攻击。   

##### HTTP 首部注入
如以下首部，Web 应用有时会把从外部接收到的数值,赋给响应首部字段 Location 和 Set-Cookie：
```html
Location: http://www.example.com/a.cgi?q=12345
Set-Cookie: UID=12345
```
用户本应输入 12345，却在后面加入了 `%0D%0A` （代表 HTTP 报文中的换行符），使请求接收方会执行 `Set-Cookie` 操作。HTTP 首部注入攻击有可能会造成以下一些影响：
1. 设置任何 Cookie 信息
2. 重定向至任意 URL
3. 显示任意的主体(HTTP 响应截断攻击)

##### HTTP 响应截断
HTTP 响应截断攻击是用在 HTTP 首部注入的一种攻击。攻击顺序相同,但是要将两个 `%0D%0A%0D%0A` 并排插入字符串后发送。利用这两个连续的换行就可作出 HTTP 首部与主体分隔所需的空行了,这样就能显示伪造的主体,达到攻击目的。这样的攻
击叫做 HTTP 响应截断攻击。
```html
%0D%0A%0D%0A<HTML><HEAD><TITLE>之后,想要显示的网页内容 ..
```

常见 Web 攻击手段还有一些，比如跨站点请求伪造（Cross-Site Request Forgeries, CSRF），之后会对此进行介绍，并比较它和跨站脚本攻击 XSS 之间的异同。

### _References_:
- [1]: [Web安全入门之常见攻击](https://zhuanlan.zhihu.com/p/23309154)
- [2]: [防止SQL注入的5种方法](https://my.oschina.net/MiniBu/blog/270521)
- [3]: [前端安全系列（一）：如何防止XSS攻击](https://tech.meituan.com/2018/09/27/fe-security.html)
- [4]: 《图解HTTP》 上野宣(日) 著 于均良 译
- [5]: [MySQL的SQL预处理（Prepared）](https://www.cnblogs.com/geaozhang/p/9891338.html)
- [6]: [mybatis官方文档——参数](http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html#Parameters)
