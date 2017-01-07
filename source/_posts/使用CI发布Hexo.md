---
title: 使用CI发布Hexo
date: 2017-01-07 13:34:08
tags:
categories: 随笔
---

**测试使用CI发布Hexo**

从此刻开始本站开始使用[travis-ci](https://travis-ci.org)发布。
博客已有两个月没有更新内容了，年底各项事宜又赶上上冲刺，这一个月忙的焦头烂额。
整理了一点东西准备发布也就放弃了，整个发布都得手工执行，今年起始的目标就有使用CI让开发更高效。
首先在博客上推行了，主要因为比较简单，东西都是现成的配置一下就能用。
网上找了一下示例还是比较多的，我参考了[这里](http://blog.csdn.net/woblog/article/details/51319364)，同时调整了`.travis.yml`, like this:

```
anguage: node_js
node_js: stable

# S: Build Lifecycle
install:
  - npm install -g hexo
  - npm install -g hexo-cli
  - npm install

script:
  - hexo clean
  - hexo g
  - hexo d

after_script:
  - cd ./public
  - git init
  - git config user.name "zahnghTK"
  - git config user.email "510223064@qq.com"
  - git add .
  - git commit -m "Update docs"
# E: Build LifeCycle

branches:
  only:
    - hexo
env:
 global:
   - GH_REF: github.com/zhanghTK/zhanghTK.github.io.git
```

好啦，本篇文章就到这里，2017起始，容我水一文。
