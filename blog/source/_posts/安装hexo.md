---
title: 安装hexo
date: 2018-06-27 12:44:00
tags: 随记
categories: 新鲜事物
---
## 前
很久之前就想写博客，之前也通过github安装过hexo。不过之前安装的时候有些问题，到时在我换电脑之后博客就打不开了，只能重新安装。这次的hexo之旅解决了之前hexo换电脑后的尴尬。在写这篇博客之前很感谢[知乎大神](https://www.zhihu.com/question/21193762/answer/79109280)提供的方案。
****
## 开始

### 博客搭建
> 主要的设计思路就是，在github的仓库上创建两个分支
> * blog 用来存放hexo的静态网页（相当于后台）
> * master 用来存放hexo编译生成的网站文件（相当于前端）

* 在github上创建仓库，取名为 xxx.github.io(xxx为你的github名称);
* 通过sourceTree把github上的仓库clone到本地；
* 创建一个分支blog，默认有一个master
* 在本地仓库地址的根目录创建一个空目录，取名hexo（这个就是以后博客的根目录）
* 通过终端cd到刚刚创建的目录下，依次执行
    ```
    //安装hexo
    npm install hexo
    //初始化hexo
    hexo init
    //安装与github的桥接文件
    npm install hexo-deployer-git --save
    ```
* 修改_config.yml中的deploy参数，例如：
    ```
    deploy:
        type: git
        repository: https://github.com/xxx/xxx.github.io.git #记得把xxx替换成你的github名字
        branch: master
    ```
    > 这样就表示你通过 hexo g -d 部署生成的文件会直自动同步到你的master分支上去
* 在blog分支下把改动的内容推送到github上去
* 执行 hexo g -d 生成网站并且部署到github上  
* over
### 日常操作

* 在本地修改之后，推送到github上
* 在终端执行 hexo g -d 重新部署 到master分支上

### 更换电脑

* 从github上clone下blog分支
* 在本地hexo（博客根文件夹）文件夹下通过终端依次执行：
    ```
    //安装hexo
    npm install hexo
    //安装与github的桥接文件
    npm install hexo-deployer-git --save
    ```
    > 注意：不需要hexo init 
    >
    > 如果你安装了主题，那么你需要重新在博客根文件夹下安装主题。

## 后记
到目前为止，你已经搭建了一个完美的可以移植的hexo博客了，当然页面很丑。

> 可以通过[hexo主题](https://hexo.io/themes/)来选择一个漂亮的主题（当然主题配置也会有很多坑）

> 推荐主题：
* [NexT是exo优雅而强大的主题](https://github.com/theme-next/hexo-theme-next)
* [exo的一个简单＆美丽＆快速的主题](https://github.com/Molunerfinn/hexo-theme-melody)(我目前使用的主题)


只是我在搭建好个人博客之后写的第一篇博客，有很多地方写得不优雅，希望大家多海涵。

在以后的日子里我会经常更新博客的。
> 千里之行始于足下

