---
title: 常见Web前端攻击小结
date: 2019-03-06 22:07:22
tags:
---

小结一下后端工程中比较常见的前端攻击以及对应的防范方法。

### 1. SQL 注入
SQL注入，指将前端通过表单或输入提交，将部分SQL命令注入到后端执行SQL中，达到修改后端逻辑目的的一种攻击方式。通常发生在后端SQL拼接时未对前端传入参数进行校验过滤的场景。
#### 1.1 SQL 注入原理
如后端有SQL：
```sql
SELECT * FROM `MyTable` WHERE `id`="{0}"
```
此时，如果前端传入 `1" OR TRUE`，拼接后SQL为
```sql
SELECT * FROM `MyTable` WHERE `id`="1" OR TRUE
```
此时将所有数据全部取出。如果后端SQL是`DELETE`，影响更是不可估量。
#### 1.2 SQL 注入的防范
防范思路主要有2种：
1. SQL预编译，后传参
2. 参数校验（前端或后端）
既然SQL注入发生在未对前端传入数据的


### 2. XSS 跨站脚本攻击
全称跨站脚本攻击，是将恶意代码植入用户页面，达到攻击目的。

### 3. CSRF(Cross Site Request Forgery) 跨站请求伪造
跨域访问

#### _References_:
- [1]: [Web安全入门之常见攻击](https://zhuanlan.zhihu.com/p/23309154)
- [2]: [防止SQL注入的5种方法](https://my.oschina.net/MiniBu/blog/270521)