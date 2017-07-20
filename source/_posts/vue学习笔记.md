---
title: vue学习笔记
date: 2016-11-29 21:04:00
tags:
  - vue
categories: 学习笔记
---

# 前言

最近由于项目需要，重新研究起了vue，说到vue，在几个月前就间歇性或多或少有过接触，因为当时用惯了react，有点不习惯vue的模式放弃了，在vue2.0出来后，以及weex的推动，vue的关注度又有了很大的提高，加之项目比较迫切，本来是几个月就该完工的项目，因为种种原因，加上研究生学习繁忙，以及react版本的代码过于复杂，导致后期维护麻烦，遂准备使用vue和apollo进行项目的重写，这篇文章是vue的学习记录

# 基础
## 上下文问题
注意，不要在实例属性或者回调函数中（如 `vm.$watch('a', newVal => this.myMethod())`）使用 **箭头函数**。因为箭头函数绑定父上下文，所以 this 不会像预想的一样是 Vue 实例，而是 `this.myMethod` 未被定义。

## 缩写
`v-bind`:
```javascript
<!-- 完整语法 -->
<a v-bind:href="url"></a>
<!-- 缩写 -->
<a :href="url"></a>
```
`v-on`:
```javascript
<!-- 完整语法 -->
<a v-on:click="doSomething"></a>
<!-- 缩写 -->
<a @click="doSomething"></a>
```
## 计算属性
在模板中使用绑定表达式很方便，但是只能用于简单的操作，在模板中放入太多的逻辑会让模板变得难以维护，所以对于任何复杂逻辑，都应该使用 **计算属性**

## 计算缓存vsMethods
不经过计算属性，我们可以在 method 中定义一个相同的函数来替代它。然而 **不同的是计算属性是基于它的依赖缓存。** 计算属性只有在它的相关依赖发生改变时才会重新取值。

## Scoped CSS
1. **A child component's root node will be affected by both the parent's scoped CSS and the child's scoped CSS**
2. Partials are **not** affected by scoped styles.
3. **Scoped styles do not eliminate the need for classes.** Due to the way browsers render various CSS selectors, `p { color: red }` will be many times slower when scoped (i.e. when combined with an attribute selector). If you use classes or ids instead, such as in `.example { color: red }`, then you virtually eliminate that performance hit. [Here's a playground](http://stevesouders.com/efws/css-selectors/csscreate.php) where you can test the differences yourself.
4. **Be careful with descendant selectors in recursive components!** For a CSS rule with the selector `.a .b`, if the element that matches `.a` contains a recursive child component, then all `.b` in that child component will be matched by the rule.
`// todo`
