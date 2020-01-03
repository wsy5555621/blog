---
title: node服务中遇到的后端问题
date: 2019-10-02 12:55:19
tags: 
- node
- alphanumeric
- big int
---
项目中使用nodeJs写web时，碰上了一些问题，花费了一些时间解决，记录一下以防再掉坑里。
<!--more-->
我们使用nodeJs实现了一个web服务，对接了后端服务的网关，并且向前端返回静态资源。
## nodeJs中的大数
首先是大数的问题。对接用户中心时，我们的node服务需要给前端返回用户id。然而，在使用此用户id去查询时，却发现查不到此用户的内容。在定位过程中，发现此用户的id甚至不在数据库中。经过一番定位，发现是我们的node服务在返回真实Id时，丢失了精度。

``` javascript
> 409119721175122920
< 409119721175122940 // 大数丢失精度

// 最大安全整数
// https://www.zhihu.com/question/29010688
> 2**53 - 1
< 9007199254740991
```
一般情况下，前后端应当约定好，涉及诸如此类可能存在大数的情况下使用字符串进行交互。
[bignumber](https://www.npmjs.com/package/bignumber.js)
[json-big-int](https://www.npmjs.com/package/json-bigint)
## 签名按字典序
字典排序（lexicographical order）是一种对于随机变量形成序列的排序方法。其方法是，按照字母顺序，或者数字小大顺序，由小到大的形成序列。
Js的sort默认采用字典序。
字典排序

http://www.codedq.net/blog/articles/395.html

## nginx开启gzip