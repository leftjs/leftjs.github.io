---
title: vue学习笔记
date: 2016-11-29 21:04:00
tags:
  - vue
categories: 学习笔记
---

# 前言

最近由于项目需要，重新研究起了vue，说到vue，在几个月前就间歇性或多或少有过接触，因为当时用惯了react，有点不习惯vue的模式放弃了，在vue2.0出来后，以及weex的推动，vue的关注度又有了很大的提高，加之项目比较迫切，本来是几个月就该完工的项目，因为种种原因，加上研究生学习繁忙，以后react版本的代码过于复杂，导致后期维护麻烦，遂准备使用vue和apollo进行项目的重写，这篇文章是vue的学习记录

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

## class与style绑定
1. 对象语法
```javascript
<div v-bind:class="{ active: isActive }"></div>
data: {
  isActive: true,
}
```
2. 数组语法
```javascript
<div v-bind:class="[activeClass, errorClass]">
data: {
  activeClass: 'active',
  errorClass: 'text-danger'
}
```

## 条件渲染
- `v-if`是一个指令，需要将他添加到一个元素上，如果需要切换多个元素可以在多个元素的外面包裹一个 `<template>`元素，并在上面使用 `v-if`，**最终的渲染结果中不包含它。**
- `v-show`和 `v-if`的使用方式基本相同，差别在于:
  1. `v-if` 是真实的条件渲染，因为它会确保条件块在切换当中适当地销毁与重建条件块内的事件监听器和子组件。
  2. `v-if` 也是 **惰性** 的：如果在初始渲染时条件为假，则什么也不做——在条件第一次变为真时才开始局部编译（编译会被缓存起来）。
  3. 相比之下， `v-show` 简单得多——元素始终被编译并保留，只是简单地基于 **CSS 切换** (`display: none`)。
- `v-else`紧跟在 `v-if`或者 `v-show`后面才用意义

## 列表渲染
1. 基本用法
```JavaScript
<ul id="example-1">
  <li v-for="item in items">
    {{ item.message }}
  </li>
</ul>
var example1 = new Vue({
  el: '#example-1',
  data: {
    items: [
      {message: 'foo' },
      {message: 'Bar' }
    ]
  }
})
```
2. of 替代 in 作为分隔符，因为它是最接近 JavaScript 迭代器的语法
```JavaScript
<div v-for="item of items"></div>
```
3. 如同 `v-if` 模板，你也可以用带有 v-for 的 <template> 标签来渲染多个元素块
```JavaScript
<ul>
  <template v-for="item in items">
    <li>{{ item.msg }}</li>
    <li class="divider"></li>
  </template>
</ul>
```
4. `v-for` 可以通过一个独享的属性来迭代
```JavaScript
<ul id="repeat-object" class="demo">
  <li v-for="value in object">
    {{ value }}
  </li>
</ul>
new Vue({
  el: '#repeat-object',
  data: {
    object: {
      FirstName: 'John',
      LastName: 'Doe',
      Age: 30
    }
  }
})
```
可以提供第二个的参数为键名：
```JavaScript
<div v-for="(value, key) in object">
  {{ key }} : {{ value }}
</div>
```
第三个参数为索引：
```JavaScript
<div v-for="(value, key, index) in object">
  {{ index }}. {{ key }} : {{ value }}
</div>
```
5. Scoped CSS

 - A child component's root node will be affected by both the parent's scoped CSS and the child's scoped CSS
 - Partials are not affected by scoped styles.
 - Scoped styles do not eliminate the need for classes. Due to the way browsers render various CSS selectors, p { color: red } will be many times slower when scoped (i.e. when combined with an attribute selector). If you use classes or ids instead, such as in .example { color: red }, then you virtually eliminate that performance hit. Here's a playground where you can test the differences yourself.
 - Be careful with descendant selectors in recursive components! For a CSS rule with the selector .a .b, if the element that matches .a contains a recursive child component, then all .b in that child component will be matched by the rule.

6. // todo
