---
title: 前端项目部署nginx基础配置参考
date: 2020-01-16 19:46:21
tags:
  - nginx
---

前段时间，面试一些前端同学的时候，发现大部分对开发完成之后的部署流程都不太了解。听到最多的回答，都是npm run build之后交给后端或者前端负责人就可以了。虽然部署的过程不一定有机会参与，但是一定要做到心中有数，不然怎么能做到负责一整个前端项目呢？

趁着有空，简单写下前端部署时候用到的最基础的nginx配置参考。这里以vue的spa项目作为例子。

## 配置分类
根据vue-router的mode，分成两种情况：

- hash
- history

同时根据部署的路径，也分成两种情况：

- 根路径
- 非根路径

我们假设dist文件夹放在/data/www/demo/dist, server_name是localhost。
<!-- more -->
### hash模式（根路径）
**nginx 配置**：
```
http {
  server {
    listen 80;
    server_name localhost;
    location / {
      root /data/www/demo/dist/;
      index index.html;
    }
  }
}
```

### hash模式（非根路径）
**nginx 配置**：
```
http {
  server {
    listen 80;
    server_name localhost;
    location ^~ /demo {
      alias /data/www/demo/dist/;
      index index.html;
    }
  }
}
```
**vue.config.js配置**：
```js
module.exports = {
  publicPath: '/demo'
}
```

### history模式（根路径）
**nginx 配置**：
```
http {
  server {
    listen 80;
    server_name localhost;
    location / {
      root  /data/www/demo/dist/;
      index index.html;
      try_files $uri $uri/ /index.html;
    }
  }
}
```

### history模式（非根路径）
**nginx 配置**：
```
http {
  server {
    listen 80;
    server_name localhost;
    location ^~ /demo {
      alias /data/www/demo/dist/;
      index index.html;
      try_files $uri $uri/ /demo/index.html;
    }
  }
}
```
**vue.config.js配置**：
```js
module.exports = {
  publicPath: '/demo'
}
```
**router配置**：
```js
const router = new VueRouter({
  mode: 'history',
  base: 'demo',
  routes // (缩写) 相当于 routes: routes
})
```

## 结语
不要把**术业有专攻**作为不愿走出舒适圈的借口，与诸君共勉。
