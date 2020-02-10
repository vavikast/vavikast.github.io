---
title: 基于Go博客wblog的理解和修改
catalog: true
date: 2020-02-07 12:32:03
subtitle:
header-img: "/img/article_header/article_header.png"
tags:
- golang
- gin
- gorm
catagories:
- golang
updateDate: 2020-02-07 12:32:03
---

基于Go博客wblog的理解和修改。

### 初衷

​		自学Go语言已经一段时间，想通过博客更深入理解go语言。最终通过Gin语言定位了[wblog博客框架]( https://github.com/wangsongyan/wblog )。wblog是基于基于gin+gorm开发的个人博客项目。

​		学习别人的博客是一个抓狂的过程，不仅要疯狂学习扩展的知识，比如gin框架，gorm，还要理解原作者的思想和构建过程。

​		原项目仅做了简单的英文注释。我则根据原项目增添了很多自己理解的注释和说明，方便其他后来人学习参考。同时更新原项目依赖，可以一键运行。

### 修改

1. 增加了中文注释，更多的是我对原项目的理解，方便其他人理解和学习。
2. 使用go module替代了govendor依赖管理包，更方便目前的运行方式。
3. 更新了代码中的依赖包，比如session中的代码。
4. 将默认使用的数据库修改为Mysql。并将时间格式设置为charset=utf8mb4&parseTime=True&loc=Local
5. 修改了其中部分显示内容。
6. 修改后自己保存的仓库为[gingorm]( https://github.com/vavikast/gingorm )。

### 说明

1. 数据库部分如果使用loc=Asia/Shanghai无法运行，所以修改为loc=Local。
2. 作者在前端页面设计增加了很多脚本函数，所以要认真看。
3. 没有理解到博客post与页面page具体的用意是什么。
4.  backup和restore只是针对了sqllite内容做的备份与恢复。
5. 注册即为是管理员，所以记得修改代码。
6. 注册后无法自动跳转到管理员页面。
7. 网站的安全性，有待验证。

### 运行

```
git  clone  https://github.com/vavikast/gingorm
cd gingorm 
go mod tidy
go run main.go
默认运行的端口是8090
```



### 补充

​	想补充一下我对这个博客项目的想法 ，原项目是非常优秀的，作为只会哔哔的人，我想说下它的问题，只是拙见，原作者莫怪。

1. 博客设计功能不够丰富。

   主要是管理操作页面有点简单，模块关联性较弱，功能简单。

2. 博客设计的过于复杂。

   1. 底层数据结构

      底层代码写的比较复杂，代码间的调用较乱，需要通篇看，才能看到各部分之间的联系。特别是struct的内容的引用和数据库的调用。

   2. 前端功能调用

      前端页面嵌入了很多脚本，也有点难以理解。

### 后记

​	    自己能动手就莫哔哔，还是要多动手，多练习。根据前辈们的优秀代码，扩展自己想要的功能。



### 	 加油啊，Felix！



