---
layout: post
title: github pages搭建博客
category: "记录"
---

###怎样搭建github pages(通过git命令行，而不是auto generator pages)###

##### 用auto generator只能选github提供的几个官方主题了，所以直接直接建更方便自由。关于怎么注册github，以及创建你的repo，问度娘就可以了

*1、make a fressh clone your repo*
> git clone http://username.github.io/yourprojectname

*2、create a gh-pages*
> git checkout --orphan gh-pages

*3、del the default file*

*4、copy other theme to the gh-pages*
> git clone others theme repo

*5、最后配置_config.yml，然后保存修改提交并push到服务器*

> git add . 

> git commit 

> git push origin gh-pages

