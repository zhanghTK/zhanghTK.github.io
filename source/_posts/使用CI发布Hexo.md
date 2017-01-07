---
title: 使用CI发布Hexo
date: 2017-01-07 13:34:08
tags:
categories: 随笔
---

**测试使用CI发布Hexo**

从此刻开始本站开始使用[Travis-CI](https://travis-ci.org)发布。
博客已有两个月没有更新内容了，年底各项事宜又赶上上冲刺，这一个月忙的焦头烂额。
整理了一点东西准备发布也就放弃了，整个发布都得手工执行，今年起始的目标就有使用CI让开发更高效。
今天在博客上实践了一下持续集成，一口气把之前整理的几篇文章都发布了，整个流程还是简单了不少。


网上关于Travis-CI的说明还是比较多的，我主要参考了[这里](http://blog.csdn.net/woblog/article/details/51319364)。

但是我掉了一个坑，我使用的主题是[nexT](http://theme-next.iissnan.com/)，直接从Github上clone得到并做了自定义，同时没有删除next主题文件夹内的`.git`文件夹。所以博客的版本控制没有直接管理next主题文件夹，在提交的时候整个next主题文件夹就没有提交上去，所以不能正确生成页面文件，导致最终网站的所有页面都成了空白页面。

解决办法很简单，删除themes/next的.git和.gitignore，加入版本控制提交就可以了。

同时我还根据报错调整了`.travis.yml`, like this:

```yaml
anguage: node_js
node_js: stable

install:
  - npm install -g hexo
  - npm install -g hexo-cli
  - npm install

script:
  - hexo clean
  - hexo g

after_script:
  - cd ./public
  - git init
  - git config user.name "zahnghTK"
  - git config user.email "510223064@qq.com"
  - git add .
  - git commit -m "Update docs"
  - git push --force --quiet "https://${GH_TOKEN}@${GH_REF}" master:master

branches:
  only:
    - hexo
env:
 global:
   - GH_REF: github.com/zhanghTK/zhanghTK.github.io.git
```

好啦，本篇文章就到这里，2017起始，容我水一文。
