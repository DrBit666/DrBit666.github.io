---

title: "手把手教你用github_page+jekyll搭建自己的个人博客"
date: 2025-03-09 18:01:00 +0800
tags: blog
categories: 技术


---

---
### 1.为什么选择使用github-page部署个人博客

```
github-page是github官方推出的由仓库构建出一个静态网站的功能。
```

它完全免费,你只需要编写.html文件并上传到你的仓库，github会自动帮你构建并部署到公共网络。

一个静态网站足以实现你想要的博客，它不需要后台，不需要数据库，仅需前端页面即可完成

---
### 2.如何使用github-page部署个人博客

1.首先,你需要有一个github账号.

...

2.使用你的github账号创建一个repository，repository仓库名格式为"{github_id}.github.io"，创建时，请务必勾选"add readme".

3.使用git clone 将你的代码克隆到本地，cd进入仓库文件夹的根目录，创建主页面

index.html
```html
<!DOCTYPE html>
<html>
<head>
    <title>我的博客</title>
</head>
<body>
    <h1>欢迎来到我的博客！</h1>
    <p>这是一个简单的个人博客示例。</p>
</body>
</html>
```

4. 提交代码并推送到origin
```zsh
git add ./
git commit -m "init"
git push
```

5.几分钟之后，使用https://{github_id}.github.io 你就可以访问你自己的github-page啦


---
### 3.如何使用jekyll设置你的博客主题

我的设备是macos 13.5.1版本，以下示例皆基于我的当前设备

1.安装homebrew并设置国内镜像

这一步就不写教程了，网上一搜有很多

2.安装ruby 3.0+，macos自带ruby2.6，但是由于jekyll4.4.1需要ruby3.0或更高版本，这里我们重新安装一下

```zsh
brew search ruby

brew install ruby@3.3

ruby -v #验证ruby版本，如果ruby版本还是2.6.n的话，去网上搜索一下怎么将ruby的可执行文件链接到@3.3版本
```

3.使用gem安装bundler和jekyll

```
gem,bundler,jekyll是什么？
```

*I-gem是ruby中的类似于包或库的概念,当你安装完ruby过后，就自动安装了gem的可执行文件*

*II-bundler既是ruby中的一个包，又是ruby的一个包管理器，有点类似于maven*

*III-jekyll也是ruby中的一个包，它的主要功能是用模版生成静态网页（这也就是我们要用到的核心功能）*

```zsh
gem install bundler jekyll  #安装

jekyll new my-awesome-site #使用jekyll初始化一个网站目录

cd my-awesome-site
bundle exec jekyll serve #使用bundle本地运行网站

#然后输入localhost:4000，你就可以看到生成的网站啦
```
4.部署到你自己的github-page上
```zsh
#将jekyll生成网站作为模版全量替换到github-page本地目录下

cp -r ./my-awesome-site/* ./{github_id}.github.io

#将代码推送到你的github origin仓库
#等待几分钟，就部署完毕了
```

---
### 4.如何使用jekyll为你生成的模版来写博客
1.写博客

在_post文件目录下新建一个.markdown文件，并在里面编辑你的文章内容。
这里的markdown文件名需满足格式(YYYY-MM-DD-title)

2.设置博客

在_config.yml文件中，你可以找到如下很多相关设置
```yml
title: xxx
email: xxx
description: >- # this means to ignore newlines until "baseurl:"
  xxx
lang: xxx
logo: xxx
```

**更多设置你可以在你选择的主题的github上找到，一般会被放到layout(布局)下，库、jekyll使用的默认主题是minima,通过修改这些设置，可以在你的网站上展示你自己独特的东西,另外，你也可以通过覆盖修改主题的默认layout实现更多定制化**

---

### 5.如何换一个其他主题
[主题清单](https://github.com/topics/jekyll-theme)
你可以在这里找到更多你想要的主题，这里用我正在使用的minimal举例

1.修改博客目录下的gem配置文件，安装主题
***Gemfile*
```zsh
gem "jekyll-theme-minimal"

```

2.通过bundle安装主题
```zsh
bundle install
```
3.修改博客配置文件_config.yaml
```yaml
theme: jekyll-theme-minimal
remote_theme: pages-themes/minimal@v0.2.0
plugins:
  - jekyll-remote-theme # add this line to the plugins list if you already have one
```

4.重新执行
```zsh
bundle exec jekyll serve #使用bundle本地运行网站
#访问localhost:4000查看效果
```
5.到此为止，minimal主题就安装完毕了，我们还需要再做一些小优化
```zsh
## 最新文章
{% for post in site.posts %}
  - [{{ post.title }}]({{ post.url }}) - {{ post.date | date: "%Y-%m-%d" }}
{% endfor %}
```
在你的index.markdown文件中加入上面这段，就可以将你的所有文章展示在你的首页啦

另外，你也可以覆盖修改它的layout(布局)和css样式来让博客有更多你自己的特点

---
### 6.写在最后

博客搭建只是写博客中最简单的一步,坚持写文章，持续内化并输出，博客才有它存在的真正意义
