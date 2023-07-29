---
layout:     post
title:      How to Deploy Slides on Github Pages
subtitle:   Use reveal.js to make magic happen 
date:       2018-06-15
author:     LiuShuo
header-img: img/post-bg-desk.jpg
catalog: true
tags:
    - Site
    - How-to
---
It's cool to have your keynote display in your blog, Github Pages has the ability to make this 
happen integrated with reveal.js.
Follow steps bellow and you can enjoy your slides on your blog too, just like [me]().

## Build from Scratch

	$ git clone https://github.com/hakimel/reveal.js.git
	$ cd reveal.js
	$ rm -rf .git
	$ git init
	$ git remote add origin git@github.com:yourname/repo-name.git
	$ git add .
	$ git commit -m "Initial commit"
	$ git push -u origin master

## Creating Branch
	$ git branch gh-pages
	$ git push origin gh-pages

All the updated files must be merged from master to gh-pages(the name can not be changed)	

The url is built from the following pattern: <br>
[github_username].github.io/[repo_name]

## Reference
- https://www.tikalk.com/posts/2013/11/05/deploy-reveal-js-slideshow-on-github-pages/
- https://www.chenhuijing.com/blog/revealjs-and-github-pages/
- https://www.youtube.com/watch?v=DUXD2q0meSw&feature=youtu.be

> 本文首次发布于 [LiuShuo's Blog](https://liushuo.me), 转载请保留原文链接.
