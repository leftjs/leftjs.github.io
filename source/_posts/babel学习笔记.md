---
title: babel核心部分笔记
date: 2017-07-20 10:56:13
tags:
  - babel
categories: 学习笔记
---

# babel核心部分笔记

## 安装方式
> 详情可以参考 [babel-cli][760ee0e9]

可以在自己电脑上全局安装babel-cli，但是更推荐在`本地`项目中安装，主要有如下两个主要原因：

1. 不同项目依赖不同版本的babel，因此可以一次更新一个babel版本
2. 无须隐式的babel依赖，使项目更易移植且容易安装。

## `.babelrc`
> 详情可参考 [.babelrc][56a5bd12]

除`callback`外的所有babel api options都可以`.babelrc`中使用，因为`.babelrc`本身是就是一个 [JSON5][25cc3d58]的序列化文件

### `.babelrc`配置方式
1. 使用 `.babelrc`文件
```json
{
  "plugins": ["transform-react-jsx"],
  "ignore": [
    "foo.js",
    "bar/**/*.js"
  ]
}
```

2. 使用 `package.json`文件
```json
{
  "name": "my-package",
  "version": "1.0.0",
  "babel": {
    // my babel config here
  }
}
```

### `env`选项
通过在 `.babelrc`中添加 `env`选项来添加不同环境下的babel配置
```json
{
  "presets": ["es2015"],
  "plugins": [],
  "env": {
    "development": {
      "plugins": [...]
    },
    "production": {
      "plugins": [...]
    }
  }
}
```

指定 `env`环境可以通过设置 `process.env.BABEL_ENV`,若前者不存在则使用 `process.env.NODE_ENV`,都不存在则使用默认值 `development`

### babel配置的搜索行为

babel首先查找被编译文件所在目录的 `.babelrc`如果不存在则上溯直到找到一个 .babelrc文件，或者一个带有 `"babel": {}`的 `package.json`

通过在 `options`中使用 `"babelrc": false` 来终止搜索行为，或者通过提供 `--no-babelrc`的cli flag



  [760ee0e9]: https://babeljs.io/docs/usage/cli/ "babel-cli"

  [56a5bd12]: https://babeljs.io/docs/usage/babelrc/ ".babelrc"

  [25cc3d58]: https://github.com/json5/json5 "JSON5"
