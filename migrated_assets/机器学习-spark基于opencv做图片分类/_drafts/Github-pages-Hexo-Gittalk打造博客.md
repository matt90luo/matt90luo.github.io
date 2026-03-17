---
title: Github pages+Hexo+Gittalk打造博客
tags:
mathjax: true 
---
本篇博客主要记录了如何使用github的 pages 以及 Hexo搭建博客，以及常见的一些问题，比如如何使用MathJax添加数学公式  
## 什么是Github Pages
首先得知道Github pages的规则：每个Github账号（比如Username）下面只能建立一个Pages，且命名必须符合这样的规则："username/username.github.io"
创建成功后，username.github.io就是你的域名（当然你可以通过别名解析绑定自己的域名）
1. 在Github建立一个新的Repository：命名为：username.github.io
2. 本地clone仓库
``` cmd
git clone https://github.com/username/username.github.io
```
3. 使用Hexo管理博客
    1. 进入clone下来的仓库目录 创建hexo分支
    ``` cmd
    git checkout -b hexo
    ```
    2. 安装hexo，以及相关扩展依赖 并初始化
    ``` cmd
    npm install -g hexo
    hexo init // 初始化
    ```
    **注意 hexo init 要求当前的目录是一个空目录可以先将目录下的内容移出 完成初始化后再移入**
    **执行 npm install**

4. hexo 主题
在githu上fork喜欢的主题, 将其作为子项目添加
git submodule add git@github.com:matt90luo/hexo-theme-next.git themes/next




## Hexo 使用教程
Node.js 安装 使用命令 `brew install node` 
使用命令 `npm install {模块名} --save` 安装插件模块
推荐安装的几个插件
1. hexo-toc
2. hexo-math
3. hexo-wordcount
4. hexo-markdown-it-tippy
5. hexo-filter-flowchart
6. hexo-filter-sequence
7. hexo-generator-json-content

在使用mathjax时，需要注意将latex格式的数学公式包裹在
`{% math %}{% endmath %}` 之内

可以去hexo官网查看相应的模块是如何配置的

``` shell
# Hexo Configuration
## Docs: https://hexo.io/docs/configuration.html
## Source: https://github.com/hexojs/hexo/

# Site
title: Blog
subtitle:
description:
author: ukiml
language: en
timezone:

# URL
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
url: https://matt90luo.github.io/
root: /
permalink: :year/:month/:day/:title/
permalink_defaults:

# Directory
source_dir: source
public_dir: public
tag_dir: tags
archive_dir: archives
category_dir: categories
code_dir: downloads/code
i18n_dir: :lang
skip_render:

# Writing
new_post_name: :title.md # File name of new posts
default_layout: draft
titlecase: false # Transform title into titlecase
external_link: true # Open external links in new tab
filename_case: 0
render_drafts: false
post_asset_folder: false
relative_link: false
future: true
highlight:
  enable: true
  line_number: true
  auto_detect: false
  tab_replace:
  
# Home page setting
# path: Root path for your blogs index page. (default = '')
# per_page: Posts displayed per page. (0 = disable pagination)
# order_by: Posts order. (Order by date descending by default)
index_generator:
  path: ''
  per_page: 10
  order_by: -date
  
# Category & Tag
default_category: uncategorized
category_map:
tag_map:

# Date / Time format
## Hexo uses Moment.js to parse and display date
## You can customize the date format as defined in
## http://momentjs.com/docs/#/displaying/format/
date_format: YYYY-MM-DD
time_format: HH:mm:ss

# Pagination
## Set per_page to 0 to disable pagination
per_page: 10
pagination_dir: page

# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
theme: icarus

# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: git@github.com:matt90luo/matt90luo.github.io.git
  branch: master

## plugin: hexo-toc
toc:
  maxdepth: 3
  class: toc
  slugify: transliteration
  decodeEntities: false
  anchor:
    position: after
    symbol: '#'
    style: header-ancho
    
## plugin: hexo-markdown-it-tippy 
markdown_it_plus:
    highlight: true
    html: true
    xhtmlOut: true
    breaks: true
    langPrefix:
    linkify: true
    typographer:
    quotes: “”‘’
    pre_class: highlightr

## plugin: hexo-math
math:
  engine: 'mathjax' # or 'katex'
  mathjax:
    src: #custom_mathjax_source
    config:
      # MathJax config
  katex:
    css: #custom_css_source
    config:
      # KaTeX config

## plugin: hexo-filter-flowchart
flowchart:
  # raphael:   # optional, the source url of raphael.js
  # flowchart: # optional, the source url of flowchart.js
  options: # options used for `drawSVG`

## plugin: hexo-filter-sequence
sequence:
  # webfont:     # optional, the source url of webfontloader.js
  # snap:        # optional, the source url of snap.svg.js
  # underscore:  # optional, the source url of underscore.js
  # sequence:    # optional, the source url of sequence-diagram.js
  # css: # optional, the url for css, such as hand drawn theme 
  options: 
    theme: 
    css_class: 
```


## 博客写作以及管理流程
1. 切换到hexo分支 使用命令 `hexo new {title}`新建博客
2. `git add .`  `git commit -m`   `git push `
3. 提交完毕后执行 `hexo generate -d` 发布博客
4. 假如换了电脑后，先clone下你github上的博客仓库，然后 `npm install -g hexo`，`npm install `和`npm install hexo-deployer-git --save`(和安装命令差不多，但是不要使用hexo init命令，因为仓库中已存在)，然后走正常博客书写发布流程即可。

## 使用gitalk作为博客评论系统
1. 首先通过 `npm i --save gitalk`  安装gitalk
2. 在主题<font color=red>layout/comment</font>新建一个<font color=red>gitalk.ejs</font> 添加如下内容
``` js
<% if (typeof(script) !== 'undefined' && script) { %>
    <link rel="stylesheet" href="https://unpkg.com/gitalk/dist/gitalk.css">
    <script src="https://unpkg.com/gitalk/dist/gitalk.min.js"></script>
        <script>
            var gitalk = new Gitalk({
              clientID:  '<%= theme.comment.gitalk.clientID %>',
              clientSecret: '<%= theme.comment.gitalk.clientSecret %>',
              repo: '<%= theme.comment.gitalk.repo %>',
              id: window.location.pathname,
              owner: '<%= theme.comment.gitalk.owner %>',
              admin: '<%= theme.comment.gitalk.admin %>',
              distractionFreeMode: '<%= theme.comment.gitalk.distractionFreeMode %>',
         })
        gitalk.render('gitalk-container')
        </script>
    <% } else { %>
        <div id="gitalk-container"></div>
    <% } %>
```
3. 在<font color=red> comment/scripts.ejs</font> 里添加
``` js
else if (theme.comment.gitalk &&
theme.comment.gitalk.owner &&
theme.comment.gitalk.repo &&
theme.comment.gitalk.admin &&
theme.comment.gitalk.clientID &&
theme.comment.gitalk.clientSecret) { %>
<%- partial('comment/gitalk', { script: true }) %>
<% } %>
```
4. 在<font color=red> comment/index.ejs</font>里面添加
``` js
else if (theme.comment.gitalk) { %>
        <section id="comments"><%- partial('comment/gitalk') %></section>
    <% } %>
```
5. 最后在主题<font color=red>_config.yml</font>中配置
``` js
gitalk: 
    clientID: 'Github Application Client ID',
    clientSecret: 'Github Application Client Secret',
    repo: 'Github repo',
    owner: 'Github repo owner',
    admin: ['Github repo collaborators'],
    distractionFreeMode: 'true'
```
其中 clientID clientSecret 需要去[github](https://github.com/settings/applications/new)申请 
 可以查看此连接 [Hexo中Gitalk配置使用教程-可能是目前最详细的教程](https://iochen.com/2018/01/06/use-gitalk-in-hexo/) 以及 [服务体验更棒的Gitalk评论系统](https://www.tiexo.cn/gitalk/) 获取详细信息


## 添加文章访问统计
