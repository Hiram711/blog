title: Python and Selenium
author: 几时西风
tags:
  - 爬虫
  - 自动化
categories:
  - 爬虫
date: 2019-07-05 10:13:00
---
## What's Selenium
> Selenium automates browsers. That's it! What you do with that power is entirely up to you. Primarily, it is for automating web applications for testing purposes, but is certainly not limited to just that. Boring web-based administration tasks can (and should!) be automated as well.

> Selenium has the support of some of the largest browser vendors who have taken (or are taking) steps to make Selenium a native part of their browser. It is also the core technology in countless other browser automation tools, APIs and frameworks.

> [以上摘自Selenium官方首页](https://www.seleniumhq.org/)

**一句话概括：Selenium可以自动控制浏览器，一般用于自动化测试，但是你想用它做其他的事情（比如爬虫、比如代替做一些无聊重复的基于网站的工作）当然也是OK的，总之根据你的需要自己鼓捣去吧。**

## Why Selenium
目前接触过的同类自动化测试工具有3个：
* [Selenium](https://www.seleniumhq.org/)
* [Puppeteer](https://github.com/GoogleChrome/puppeteer)
* [Splash](https://github.com/scrapinghub/splash)

他们各有优缺点，根据需要使用，个人的话更偏向于使用Selenium:

工具|官方文档|社区活跃度|编程语言|支持浏览器|安装|API|BUG|效率
-:|-:|-:|-:|-:|-:|-:|-:|:-
Selenium|完备易读|非常活跃，问题通常能得到解答|Java,Python,PHP|基本支持所有主流浏览器|较为简单快捷，selenium加上相应浏览器与浏览器驱动即可|丰富且易于使用|较少|阻塞式，最慢，最好利用官方提供的selenium grid集群提升效率
Puppeteer|完备易读|相对活跃|Javascript,另外虽然有个人开发的pyppeteer作为python支持，但是不建议使用|本质其实是一个无头的chrome浏览器|基于node.js，需要使用npm安装，相对复杂|相对丰富但因为加入了协程机制使用起来较为复杂|目前仍然有不少坑|由于使用协程机制，速度较快
Splash|完备易读|相对活跃|Lua，对接scrapy时非常好用|本质是一个异步js渲染服务引擎|非常简单，使用docker安装|丰富，但使用起来时需要编写lua脚本作为请求参数，略微复杂|有一些但不多，可以接受|异步，配合协程机制（比如scrapy）很快

## How
to be continued...