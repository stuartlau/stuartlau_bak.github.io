---
layout:     post
title:      "Font Awesome is Really Awesome"
subtitle:   "The web’s most popular icon set and toolkit"
date:       2018-09-26
author:     LiuShuo
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Site
---
# Font Awesome
[Font Awesome](https://fontawesome.com) is a website provides free icons.
> Get vector icons and social logos on your website with Font Awesome, the web’s most popular icon set and toolkit.

![font-awesome-collection](https://stuartlau.github.io/img/in-post/font-awesome-collection.jpg)

The latest version is 5.3 and it has 1,341 Free Icons and 2,637 Pro Icons for business usage.
 
# How To Add Icons
To insert an icon, add the name of the icon class to any inline HTML element.

The `<i>` and `<span>` elements are widely used to add icons.

All the icons in the icon libraries below, are scalable vector icons that can be customized with CSS (size, color, shadow, etc.)


To use the Font Awesome icons, add the following line inside the `<head>` section of your HTML page:

    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.7.0/css/font-awesome.min.css">

# Add Wexin Icon
I want to add Wechat Icon to my blog and share my account QR code with public, just add the 
following to my footer:

    <span class="fa-stack fa-lg">
        <i class="fa fa-circle fa-stack-2x"></i>
        <i class="fa fa-wechat fa-stack-1x fa-inverse" style="color:#1fb922"></i>
    </span>
    
Now let's see how it looks in my site
![footer-icons]({{ site.url }}/img/in-post/footer-icons.jpg)
        
# Reference
- https://fontawesome.com
- https://www.w3schools.com/icons/default.asp

> 本文首次发布于 [LiuShuo's Blog](https://liushuo.me), 转载请保留原文链接.
