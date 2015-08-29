---
layout: post
title: "迁移博客到Github"
date: 2014-11-12 23:26:28 +0800
comments: true
categories: 未分类
---


# 博客历程

[CNBlog](http://www.cnblogs.com/JavaTechLover/ "daveztong")->[waye.me](http://waye.me/ "daveztong") (closed)->[OSChina](http://my.oschina.net/javaTechLover/blog "daveztong")->Now here i am.

正如[阮一峰](http://www.ruanyifeng.com/)所说,喜欢写博客的人一半会经历三个阶段:
> 1. 第一阶段，刚接触Blog，觉得很新鲜，试着选择一个免费空间来写。
> 2. 第二阶段，发现免费空间限制太多，就自己购买域名和空间，搭建独立博客。
> 3. 第三阶段，觉得独立博客的管理太麻烦，最好在保留控制权的前提下，让别人来管，自己只负责写文章。

我想我现在处于第三个阶段吧。虽然语文学的不好，但我一直都有写博客记录自己学习的习惯，写博客证明我还在学习还在进步，不管写的真么样，但总还是有一些收获的。辗转过这么多部落,finally, i am home!

__<span style="color:red">Stop being a sentimentalist ^_^</span>__ 

ok,废话少说，还是先记录一下Octopress的安装和使用吧!

# 1.前提条件
> * 安装git
> * ruby

#2.Setup Octopress

```
git clone git://github.com/imathis/octopress.git octopress
cd octopress
```
Next, install dependencies.

```
gem install bundler
rbenv rehash    # If you use rbenv, rehash to be able to run the bundle command
bundle install
```
Install the default Octopress theme.

```
rake install
```
# 3.部署到github pages
Create a [new Github repository](https://github.com/repositories/new) and name the repository with the format `username.github.io`, where `username` is your GitHub user name or organization name.

Github Pages for users and organizations uses the master branch like the public directory on a web server, serving up the files at your Pages url `http://username.github.io`. As a result, you'll want to work on the source for your blog in the source branch and commit the generated content to the master branch. Octopress has a configuration task that helps you set all this up.

```
rake setup_github_pages
```
The rake task will ask you for a URL of the Github repo. Copy the SSH or HTTPS URL from your newly created repository (e.g. `https://github.com/daveztong/daveztong.github.io.git`) and paste it in as a response.

This will:

1. Ask for and store your Github Pages repository url.
2. Rename the remote pointing to imathis/octopress from 'origin' to 'octopress'
3. Add your Github Pages repository as the default origin remote.
4. Switch the active branch from master to source.
5. Configure your blog's url according to your repository.
6. Setup a master branch in the `_deploy` directory for deployment.

Next run:
```
rake generate
rake deploy
```
This will generate your blog, copy the generated files into `_deploy/`, add them to git, commit and push them up to the master branch. In a few seconds you should get an email from Github telling you that your commit has been received and will be published on your site.

__Don't forget__ to commit the source for your blog.

```
git add .
git commit -m 'your message'
git push origin source
```
**Note**: With new repositories, Github sets the default branch based on the branch you push first, and it looks there for the generated site content. If you're having trouble getting Github to publish your site, go to the admin panel for your repository and make sure that the master branch is the default branch.

To read more on creating your first blog post, read [Blogging Basics](http://octopress.org/docs/blogging/)


## Then you are good to go.

---
