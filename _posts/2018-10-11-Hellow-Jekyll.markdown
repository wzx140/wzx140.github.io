---
layout: post
title:  "Jekyll环境搭建与入门"
date:   2018-10-10
categories: Others
tags: Jekyll
mathjax: false
author: wzx
---

* content
{:toc}

*jekyll*是一个简单的免费的Blog生成工具，只能生成静态网页的工具，不需要数据库支持。*jekyll*可以免费部署在Github上，而且可以绑定自己的域名。




## Jekyll环境搭建
这里在**ubuntu16.04**上搭建环境，*linux*几行代码就能搞定

1. `sudo apt-get  install  build-essential`安装*gcc*，`gcc -v`、`g++ -v`检查是否安装成功

2.  *Make*一般都自带，`make -v`检查是否安装

3. `sudo apt-get install ruby`安装*Ruby*，`ruby -v`检查是否安装成功

4. `sudo apt-get install ruby-dev`这个也要安装，不然*Jekyll*无法安装

4. `sudo apt-get install rubygems`安装*RubyGems*，`gem -v`检查是否安装成功

5. `sudo gem install jekyll bundler`安装*Jekyll*

## Jekyll 主题选择
1. 点击前往[jekyll 主题官网](https://link.jianshu.com/?t=http%3A%2F%2Fjekyllthemes.org%2F)
2. 解压文件，cd到目录，执行`sudo jekyll server`

错误1：
```
Dependency Error: Yikes! It looks like you don't have jekyll-paginate or one of its dependencies installed. In order to use Jekyll as currently configured, you'll need to install this gem. The full error message from Ruby is: 'cannot load such file -- jekyll-paginate' If you run into trouble, you can find helpful resources at https://jekyllrb.com/help/!
jekyll 3.8.4 | Error:  jekyll-paginate
```

错误2：
```
Dependency Error: Yikes! It looks like you don't have jekyll-sitemap or one of its dependencies installed. In order to use Jekyll as currently configured, you'll need to install this gem. The full error message from Ruby is: 'cannot load such file -- jekyll-sitemap' If you run into trouble, you can find helpful resources at https://jekyllrb.com/help/!
jekyll 3.8.4|Error:  jekyll-sitemap
```

**解决方法**：在**Gemfile**中添加
```ruby
source "https://rubygems.org"
gem "jekyll-paginate"
gem "jekyll-sitemap"
```

## Jekyll 目录结构
* *_posts* **:** 博客内容

*  *_pages* **:** 其他需要生成的网页，如About页

*  *_layouts* **:** 网页排版模板

*  *_includes* **:** 被模板包含的HTML片段，可在_config.yml中修改位置

*  *assets* **:** 辅助资源 css布局 js脚本 图片等

*  *_data* **:** 动态数据

*  *_sites* **:** 最终生成的静态网页

* *_config.yml* **:** 网站的一些配置信息

*  *index.html* **:** 网站的入口

## Jekyll 教程资源
1. [leach_chen](https://www.jianshu.com/p/9f71e260925d)

2. [jmcglone](http://jmcglone.com/guides/github-pages/)

3. [官方文档](https://jekyllrb.com/docs/)
