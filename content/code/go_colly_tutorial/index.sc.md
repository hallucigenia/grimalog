---
title: "Go爬虫框架colly实战"
date: 2020-03-17T15:47:56+08:00
draft: true

categories: ['Code']
tags: ['Colly', 'Golang']
author: "Fancy"
---
作为从Python过来的人，想必对用`Scrapy``Beautiful Soup`都不能再熟悉，甚至于代理池以及做分布式（后期有机会会整理分享下我个人的实现），以满足工作业务需求以及个人生活兴趣，不至于得心应手也是勉强够用了，近两个月来的Golang学习，对于基础和后端了解比较多，这次就用colly爬虫练练学习下吧。

<!--more-->
[colly](http://go-colly.org)是Golang排名第一的爬虫框架，支持分布式，高效，官网文档和示例代码明了，清晰的API。我们这次以常用的豆瓣为例，实践下colly的使用。

安装gocolly及其相关包依赖：

```bash
$ go get -u github.com/gocolly/colly/...
```

colly 的主体是 Collector 对象，使用colly必先初始化Collector
```go
c := colly.NewCollector()
```

 在初始化之后你可以使用不同类型的回调函数，其调用顺序如下：

1. `OnRequest` **请求前**调用
2. `OnError` **请求中**发生错误调用
3. `OnResponse`接收到响应后调用
4. `OnHTML`在 *OnResponse* 之后如果收到的内容为HTML则调用

5. `OnXML`在 *OnHTML* 之后如果收到的内容为HTML或XML则调用

6. `OnScraped` 在 *OnXML* 之后调用



回顾下简单实例：

```go
package main

import (
	"fmt"

	"github.com/gocolly/colly"
)

func main() {
	// colly 的主体是 Collector 对象，使用colly必先初始化Collector
	c := colly.NewCollector(
		// Visit only domains: hackerspaces.org, wiki.hackerspaces.org
		colly.AllowedDomains("hackerspaces.org", "wiki.hackerspaces.org"),
	)

	// On every a element which has href attribute call callback
	c.OnHTML("a[href]", func(e *colly.HTMLElement) {
		link := e.Attr("href")
		// Print link
		fmt.Printf("Link found: %q -> %s\n", e.Text, link)
		// Visit link found on page
		// Only those links are visited which are in AllowedDomains
		c.Visit(e.Request.AbsoluteURL(link))
	})

	// Before making a request print "Visiting ..."
	c.OnRequest(func(r *colly.Request) {
		fmt.Println("Visiting", r.URL.String())
	})

	// Start scraping on https://hackerspaces.org
	c.Visit("https://hackerspaces.org/")
}
```



代码中导入：
```go
package main

import (
	"fmt"

	"github.com/gocolly/colly"
)

func main() {
	c := colly.NewCollector(
		colly.UserAgent("Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)"),
	)

	c.OnRequest(func(r *colly.Request) {
		fmt.Println("Visiting", r.URL)
	})

	c.OnError(func(_ *colly.Response, err error) {
		fmt.Println("Something went wrong:", err)
	})

	c.OnResponse(func(r *colly.Response) {
		fmt.Println("Visited", r.Request.URL)
	})

	c.OnHTML(".paginator a", func(e *colly.HTMLElement) {
		e.Request.Visit(e.Attr("href"))
	})

    c.OnScraped(func(r *colly.Response) {
        fmt.Println("Finished", r.Request.URL)
    })

	c.Visit("https://movie.douban.com/top250?start=0&filter=")
}

```

