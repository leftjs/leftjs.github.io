---
title: js中加号特殊用法一览
date: 2017-03-10 10:21:22
tags:
categories:
---
> js中的 `+` 有多种用途，总结如下

```javascript
// 16进制转换:
+”0xFF”;              // -> 255
// 获取当前的时间戳,相当于`new Date().getTime()`:
+new Date();
// 比 parseFloat()/parseInt()更加安全的解析字符串
parseInt(“1,000″);    // -> 1, not 1000
+”1,000″;             // -> NaN, much better for testing user input
parseInt(“010″);      // -> 8, because of the octal literal prefix
+”010″;               // -> 10, `Number()` doesn't parse octal literals
//一些简单的缩写比如： if (someVar === null) {someVar = 0};
+null;                // -> 0;
// 布尔型转换为整型
+true;                // -> 1;
+false;               // -> 0;
//其他:
+”1e10″;              // -> 10000000000
+”1e-4″;              // -> 0.0001
+”-12″;               // -> -12：
```
