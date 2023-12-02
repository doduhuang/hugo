---

title: "分析ZygoteInit"
date: "2017-08-21"
description: "分析ZygoteInit"
categories:
  - "Android"
  - "源码分析"
tags:
  - "Android"
  - "源码分析"
menu: page # Optional, add page to a menu. Options: main, side, footer

# Theme-Defined params
#thumbnail: "img/placeholder.png" # Thumbnail image
lead: "分析ZygoteInit" # Lead text
comments: false # Enable Disqus comments for specific page
#authorbox: true # Enable authorbox for specific page
pager: true # Enable pager navigation (prev/next) for specific page
toc: true # Enable Table of Contents for specific page
mathjax: true # Enable MathJax for specific page
sidebar: "right" # Enable sidebar (on the right side) per page
widgets: # Enable sidebar widgets in given order per page
  - "post_toc"
---

[TOC]

### 利用Exception()触发函数执行流程

> 新建一个项目在MainActivity的onCreate方法中执行log触发流程 log如下

```java
android.util.Log.e("xiaozhu", "onCreate: ", new Exception());
```



![image-20231201220714617](../%E5%AE%89%E5%8D%93%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90ZygoteInit.assets/image-20231201220714617-17014396415831.png)
