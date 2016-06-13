---
layout: post
title:  "SDWebImage学习笔记"
date:   2016-06-07 14:55:37
categories: animate
---

SDWebImage是iOS开发中最常用的第三方图片下载库,本文是对SDWebImage源码的学习笔记.

源码地址：https://github.com/rs/SDWebImage

主要的功能有:

- An UIImageView category adding web image and cache management to the Cocoa Touch framework (添加UIImageView的分类实现网络图片加载和缓存的功能)

- An asynchronous image downloader (异步图片下载器)

- An asynchronous memory + disk image caching with automatic cache expiration handling （异步的内存和磁盘缓存）

- Animated GIF support （支持GIF图片）

- WebP format support （支持WebP图片）

- A background image decompression  （后台图片解压缩)

- A guarantee that the same URL won't be downloaded several times （确保相同的URL不会被多次下载）

- A guarantee that bogus URLs won't be retried again and again （确保假的URL不会被重复加载）

- A guarantee that main thread will never be blocked （确保主线程不会被堵塞）

- Performances! (性能!)

- Use GCD and ARC （使用GCD和ARC）

- Arm64 support  （支持64位）

# 关于下载













