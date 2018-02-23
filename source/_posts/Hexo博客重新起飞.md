---
title: Hexo博客重新起飞
tags: [otherTec]
---
>开篇：好久没来这里倒腾了，意料之中地把hexo相关的东西忘了个精光，好在并没有浪费太多的时间，感觉还是有必要记录一下，以免浪费时间。
环境：MACOS,已经有git了

<!-- more -->

### 1.安装node.js
    这个直接在网上搜一下官方就行了，或者点这里https://nodejs.org/en/download/
### 2.安装hexo
    sudo npm install hexo-cli -g
### 3.远程克隆项目
    eg: git clone https://github.com/ccflower/blogBackup.git
### 4.进入项目，安装HEXO
    npm install hexo --save
### 5.项目HEXO初始化,安装依赖
    hexo init   (这一步可能会有提示，需要用一个空目录init，然后再把所有东西考进去)
    npm install
    hexo s (这个就可以本地看了)
### 6.发布
    hexo g
    hexo d



>参考文献：
1.http://ccflower.space/2017/06/01/hexoBlog
2.http://todd2010.github.io/2016/05/18/02-Hexo-GitHub/
3.http://ryane.top/2018/01/10/2018%EF%BC%8C%E4%BD%A0%E8%AF%A5%E6%90%AD%E5%BB%BA%E8%87%AA%E5%B7%B1%E7%9A%84%E5%8D%9A%E5%AE%A2%E4%BA%86%EF%BC%81/