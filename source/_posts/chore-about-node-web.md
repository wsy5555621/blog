---
title: node服务中遇到的后端问题
date: 2019-10-02 12:55:19
tags: 
- node
- alphanumeric
- big int
---
项目中使用nodeJs写web业务时，碰上了一些问题，花费了一些时间解决，记录一下以防再掉坑里。
<!--more-->
我们使用nodeJs实现了一个web服务，对接了后端服务的网关，并且向前端返回静态资源。
## nodeJs中的整数
首先是大整数的问题。对接用户中心时，我们的node服务需要给前端返回用户id。然而，在使用此用户id去查询时，却发现查不到此用户的内容。在定位过程中，发现此用户的id甚至不在数据库中。最后发现是我们的node服务在返回真实id时，直接返回了整型，丢失了精度。
``` javascript
> 409119721175122920
< 409119721175122940 // 大整数丢失精度
```

JS中的Number对象是采用`双精度IEEE 754 64位浮点`类型来存储的，且对整数的表示等同于双精度浮点数（64-bit）对整数的表示。它的规则如下：

| Sign(符号位) | Exponent(指数位偏移) | Fraction(有效数字) |
| :----: | :----: | :-: |
| 1 | 11 | 52 |
| - | 偏移了2^(11 - 1) - 1位 | 1.f * 2 ^ p | 


| 表达式 | 约束 | 说明 |
| :----: | :----: | :-: |
| (−1)^s × 1.f × 2^(e−1023) | 0 < e < 2047, −1022～+1023加上1023 | 规约化数值 |
| (−1)^s × 0.f × 2^(-1022)	| e = 0, f > 0, **指数不是0-1023，依旧为-1022(规定)** | 非规约化数值，用来表示无限接近0的数 |
| (−1)^s × 0	| e = 0, f = 0 | 正负0，js中它们相等 |
| NaN	| e = 2047, f > 0 | 指数位全为1，有效数字大于0，不是number |
| (−1)^s × ∞ (infinity)	| e = 2047, f = 0 | 正负无穷 |

为什么52位有效数字能表示53位的精度呢？IEEE754规定规约数的尾数第一位隐含为1，不存入比特位中。那么，1.xxxx(52个x)最多共有53位精确位，拿来做整数表示，这53个二进制1转换为10进制，值为2^53 - 1。

[`BigInt`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/BigInt)：BigInt是一种内置对象，它提供了一种方法来表示大于 2^53 - 1 的整数。这是Javascript中可以用Number表示的最大安全整数。BigInt可以表示任意大的整数。可以用在一个整数字面量后面加`n`的方式定义一个BigInt。

``` javascript
// 最大安全整数
// https://www.zhihu.com/question/29010688
> 2**53 - 1
< 9007199254740991

typeof 1n === 'bigint'; // true
```

综上，前后端应当约定好，处理此类可能存在大整数的情况下，使用字符串来进行交互。这里我们使用了[json-big-int](https://www.npmjs.com/package/json-bigint)来处理json中的大整数。

``` javascript
import { parse } from 'json-bigint'; 

fetch('/url')
  .then(res => res.text())
  .then(res => parse(res)) // 其中的大整数将被bignumber库处理
```

最后，怎么来判断大整数呢？
* 字符串位数
* ECMAScript 6中MAX_SAFE_INTEGER和MIN_SAFE_INTEGER
* [bignumber](https://www.npmjs.com/package/bignumber.js)中有更为严谨的判断方法

``` javascript
Number.isSafeInteger = function (n) {
  return (typeof n === 'number' &&
    Math.round(n) === n &&
    Number.MIN_SAFE_INTEGER <= n &&
    n <= Number.MAX_SAFE_INTEGER);
}
```

## 签名按字典序
在数字签名鉴权时，通常会要求按一个字典序来排列参数。字典排序（lexicographical order）是一种对于随机变量形成序列的排序方法。其方法是，按照字母顺序，或者数字小大顺序，由小到大的形成序列(**0-9A-Za-z**)。而Js的sort默认采用字典序。


``` javascript
[1,2,5,10, 'A', '', 'ba','ab', 'aa'].sort(); // ["", 1, 10, 2, 5, "A", "aa", "ab", "ba"]
```
* 多个字符串按字典序排列
* 一个字符串按字典序排列
* 全排列

## nginx开启gzip
nginx默认是关闭gizp的。
``` bash
server {
  listen      80;
  server_name .example.com
  # max upload size
  client_max_body_size 75M;
  # 启用 gzip 压缩功能
  gzip on;
  # nginx做前端代理时启用该选项，表示无论后端服务器的headers头返回什么信息，都无条件启用压缩
  gzip_proxied any;
  gzip_vary on;
  gzip_http_version 1.1;
  # 哪些种类开启 gzip_types  *; 是没效果的
  gzip_types application/javascript application/json text/css text/xml image/png;
  gzip_comp_level 4;

  location /example  {
    alias /usr/local/etc/files/mysite/media_root;
  }

}
```
![](/post-images/gzip-open.png)

## JWT退出登陆
当点了退出登陆后，怎么清除客户端侧的登陆信息呢：设置空的session cookie。
``` javascript
'get /api/logout': (req, res) => {
  res.clearCookie('session_usr');
  return res.send('ok');
}
```
如果是多系统共用用户中心，在别的系统中退出，如何通知我们的系统清空用户状态呢？有一种方式是使用`<img>`向我们的系统发送一个上述请求，清空服务端的session，同时向浏览器种下空session。
