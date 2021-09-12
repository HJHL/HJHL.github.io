---
title: 基于Github Pages&Action自动化部署Hexo blog
date: 2021-09-12 23:59:38
tags:
---

现如今，部署一个博客是最简单不过的事情之一了，谷歌一搜能找到大片如Wordpress、Hexo等相关的博客部署教程。只是互联网是发展迅速的，短短的时间过去很多东西就发生了变化。本文简单记录下该博客此次的部署历程。



## 一、确定博客部署方案

### 1. 动态与静态的选择

考虑到部署动态博客，维护起来太麻烦了，需要准备服务器、域名等等乱七八糟的东西，遂选择部署**静态博客**。

这里我选择的是[hexo](https://hexo.io)。

### 2. 部署位置选择

既然选择了静态博客，肯定是首选部署在[**Github Pages**]([GitHub Pages | Websites for you and your projects, hosted directly from your GitHub repository. Just edit, push, and your changes are live.](https://pages.github.com/))上了。后面为了优化国内的访问体验再说，比如套CDN，或者部署到Gitee，再或者搞个VPS。

### 3. 部署方式的选择

唉，越活越懒，尽管`hexo d`部署的方式也很方便，但能自动化自然就选择自动化了。非常开心的是Github提供了[**Github Actions**](https://github.com/features/actions)服务。



## 二、搭建历程

### 1. 构建基础环境

安装Node.js环境和npm命令。

```bash
brew install node
```

 全局安装hexo。

```bash
npm install hexo-cli -g
```

创建一个博客站点。

```bash
hexo init blog
```

以git submodule形式添加Next主题。（选择Next主要是图个方便）

```bash
cd blog/ # 切到
git submodule add https://github.com/next-theme/hexo-theme-next.git themes/next
```

> 截止到2021-09-13，Next主题有三个repo，分别是
>
> * [next-theme/hexo-theme-next](https://github.com/next-theme/hexo-theme-next)
> * [theme-next/hexo-theme-next](https://github.com/theme-next/hexo-theme-next)
> * [iissnan/hexo-theme-next](https://github.com/iissnan/hexo-theme-next)
>
> 最新的是第一个。关于此原因，可以看这个Issue：[【必读】更新说明及常见问题 · Issue #4 ](https://github.com/next-theme/hexo-theme-next/issues/4)

再修改下站点的_config.yml文件。

```yaml
theme: next
```

> 自Hexo 5.0版本后，推荐使用`$BLOG_ROOT/_config.[theme-name].yml`来替代旧的方式。这样做的好处是更新主题不会遇到冲突问题。

### 2. 客制化

预先声明：

* 站点配置文件：`$BLOG_ROOT/_config.yml`
* 主题配置文件：`$BLOG_ROOT/_config.next.yml`（因为选用了next主题）

#### 开启暗黑模式

next主题在`v7.7.2`支持了暗黑模式，可以在启用深色模式的系统下使用，默认关闭。

```yml
darkmode: true
```

#### 添加百度&谷歌收录，方便SEO。

```yml
baidu_site_verification: <you-baidu-identify-code> # 替换成自己的
google_site_verification: <you-google-identify-code> # 替换成自己的
```

#### 添加百度站点统计

```yml
baidu_analytics: <you-baidu-analytics-app-id>
```

#### 添加文章访问量统计

基于[leancloud](https://leancloud.cn)统计文章访问量。

由于一个安全问题，现在添加较为麻烦，可以参考以下两篇博文：

* [Hexo Next leancloud文章阅读次数配置](https://www.jianshu.com/p/e0a719bac963)
* https://blog.garryde.com/archives/48665.html

注意：这里需要安装插件hexo-leancloud-counter-security，请不要直接用https://github.com/theme-next/hexo-leancloud-counter-security，因为它存在bug。可以使用这个：https://github.com/799953468/hexo-leancloud-counter-security。

#### Github Action自动化部署

可以抄我这里的配置：[HJHL.github.io/hexo-deploy.yml at main · HJHL/HJHL.github.io](https://github.com/HJHL/HJHL.github.io/blob/main/.github/workflows/hexo-deploy.yml)

它会在每次`main`分支有新提交时，自动触发部署操作。

博客部署到当前Repo的`blog`分支，因此需要在Repo的pages中，将branch设为`blog`。

主题，需要在repo的settings中，添加以下**三个**`secrets`。

* SSH_PRIVATE_KEY：你的Github或当前Repo的私钥，需要具有写权限
* GIT_EMAIL：部署博客时，提交commit用到的email
* GIT_NAME：部署博客时，提交commit的name



TBD……
