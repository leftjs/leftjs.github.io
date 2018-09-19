# 个人博客，基于博客引擎 [Hexo][045c4255] 和 [Next][1c61b435] 主题

[045c4255]: https://hexo.io/zh-cn/ "Hexo"
[1c61b435]: https://github.com/theme-next/hexo-theme-next "Next"

## 自定义 npm 脚本，方便项目部署和在新设备上配置

```json
"scripts": {
  "preinstall": "if [ ! -d \"themes/next\" ]; then git clone https://github.com/theme-next/hexo-theme-next themes/next && cp _next_config_backup.yml themes/next/_config.yml; fi",
  "backup": "cp themes/next/_config.yml ./_next_config_backup.yml",
  "push": "git add . && git commit -m \"backup at `date`\" && git push origin backup",
  "deploy": "npm run backup && npm run push && hexo d --g"
},
```

## 使用方法

1. 克隆本项目

```bash
export YOUR_NAME=leftjs && git clone -b backup https://github.com/leftjs/leftjs.github.io.git $YOUR_NAME.github.io && cd $YOUR_NAME.github.io && rm -rf .git && git init && git remote add origin "https://github.com/$YOUR_NAME/$YOUR_NAME.github.io.git"
```

2. 改变 `_config.yml` 中的部署地址

```
deploy:
  type: git
  repository: git@github.com:[YOUR_NAME]/[YOUR_NAME].github.io
  branch: master
```

## NOTE

1. 自行在 github ssh settings 页面里面添加本机的 ssh keys
2. git 相关问题请熟悉 git 使用以及 hexo deploy 流程
