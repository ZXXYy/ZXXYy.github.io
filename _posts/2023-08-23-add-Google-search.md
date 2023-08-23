---
title:      "让Google搜索到你的Github网页（使用Jekyll搭建）"
date:       2023-08-23 09:00:00
author:     zxy
math: true
categories: ["Coding", "Misc"]
tags: ["misc", "web"]
post: true
---

## 1. 检查网页能否被Google搜索到

在谷歌搜索栏中输入`site:https://xxxx.github.io`，查看网页是否已经被Google收录。
在这里，我在搜索栏中输入`site:https://zxxyy.github.io/`，显示的结果是尝试使用Google Search Console，则意味着没有被收录。
![try](/assets/img/in-post/2023-08-23-google/try-google.png)

## 2. Google网站所有权认证

- 点击网页[Google Search Console](https://search.google.com/search-console/about) 进行配置
- 点击网址前缀添加（网域方式验证方式单一），提交博客网址，点击继续跳转到验证界面。
![prefix](/assets/img/in-post/2023-08-23-google/prefix.png)
- 此处，我们采用“将 HTML 文件上传至您的网站”的方式进行验证。点击下载文件，把下载的文件添加到Github.io网页的根目录下。
![verification](/assets/img/in-post/2023-08-23-google/verification.png)

    ```
    .
    ├── CHANGELOG.md
    ├── LICENSE
    ├── README.md
    ├── _config.yml
    ├── _data
    ├── _layouts
    ├── _posts
    ...
    ├── assets
    ├── Gemfile
    ├── google5667ab00ac76dc5d.html
    ```

把所有文件push到GitHub上。

- 在 Google Search Console的验证页面点击验证

![success](/assets/img/in-post/2023-08-23-google/success.png)

认证成功后，Google可能需要几天的时间处理，处理完成后，我们可以在Google Search Console看到网页的概述信息。

## 3. 创建sitemap

站点地图(Site Map)存储站点的结构，例如有多少页面，以及每个页面的URL是什么等。Google可以通过这个文件知道我们网站的结构。
GitHub可以自动生成sitemap：

- 在`Gemfile`文件中加入`gem "jekyll-sitemap"`
- 在`_config.yml`文件中加入

    ```
    plugins: 
    - jekyll-sitemap
    ```

- 运行`bundle exec jekyll serve`，在网站根目录(`_site/`)下`sitemap.xml`文件会自动生成。
再把修改的内容push到GitHub，通过查看`https://zxxyy.github.io/sitemap.xml`可以看到生成的sitemap。

- 在Google Search Console中添加站点地图`https://zxxyy.github.io/sitemap.xml`，这样Googlebot就可以分析你的网站了。
![success](/assets/img/in-post/2023-08-23-google/sitemap.png)
一般上，这也要花上好几天的时间，我们才可以在谷歌中搜到自己的网站。在如下的页面中可以查看进度。
![patience](/assets/img/in-post/2023-08-23-google/patience.png)


## Reference
1. [Make your Jekyll Github Pages appear on Google search result (2 steps)](https://victor2code.github.io/blog/2019/07/04/jekyll-github-pages-appear-on-Google.html)
2. [让Google搜索到GitHub Pages](https://saowu.top/blog/4tCVcic30/#&gid=1&pid=2)
3. [Google收录Github Pages静态网页](https://charlesqueen.github.io/2021/08/13/problem-of-next-2/)