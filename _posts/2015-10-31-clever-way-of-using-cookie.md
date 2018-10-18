---
layout: post
title: 应对反爬虫之换 Cookie
date: 2015-10-31T16:07:58+08:00
tags: 
  - 爬虫
  - Python

---


最近想玩一玩数据分析，却苦于没有数据。所以想写写爬虫抓些数据来分析分析。想来想去，把目标瞄准了新浪微博。虽然现在微博已经不太行了，但是毕竟UGC内容很多，并且都是短文本，感觉比较好处理些。

说搞就搞，通过抓包分析移动版微博，很容易就知道新浪微博的api接口（获取粉丝列表，获取微博列表），更爽的是，这些接口不需要登陆态，并且返回的都是json数据。本来我的目的就是分析数据，正好也不想花费太多精力在爬虫上。基于python的request库很快写好了代码，一跑，嘿嘿，一下子抓了几千条数据下来，但是跑不到几分钟之后，就被封掉了。。- -!

怎么搞？我也是爬虫菜鸟。记忆中要减少被封的几率，只有1. 换ip，2. 降低爬取频率两个办法。所以我在代码里加了些sleep，降低了些爬取的频率。再跑一次，这次坚持久了些。但是跑了5，6分钟后，还是被封了。

出师未捷身先死，跑个几分钟，被封几小时，这爬虫是搞不下去了。只好上网Google一番如何应对反爬虫。幸运的是找到下面这篇文章《记搜狗微信号搜索反爬虫》。作者提到他在探索搜狗微信公众号反爬虫时，分析出服务器是通过Cookie来做反爬的，因此他维护了一个Cookie池，定期去更新，爬虫则从Cookie池中取Cookie。

于是我也尝试了一把，把cookie换掉，再访问一次，发现竟然又能访问了。看来新浪微博的反爬虫机制也是基于Cookied的，并非基于ip。这办法可行！接下来就很简单了，搞一个Cookie生成器，定期往Cookie池里塞Cookie，爬虫每次爬取前都从Cookie池里取Cookie出来使用，如果发现使用该Cookie不能访问了，则将其从Cookie池里删掉。下面看看代码：

**Cookie生成器**：每隔20秒往服务器发送不带Cookie的head请求，解析回包的set-cookie的值，提取出Cookie并保存到CookieCollector中。
<img src="/assets/images/clever-way-of-using-cookie/cookie-generator.png" alt="Cookie生成器" title="Cookie生成器" width="800" />

**爬虫**：爬取前取Cookie，如果发现Cookie不可用，则将该Cookie从CookieCollector中删除。
<img src="/assets/images/clever-way-of-using-cookie/cookie-pool.png" alt="cookie池" title="cookie池" width="800" />

**Cookie池**：这里我采用了Redis的有序集合来实现，score值则为获取到Cookie的时间，这是为了让爬虫每次取Cookie都先取最旧的Cookie。
<img src="/assets/images/clever-way-of-using-cookie/crawler.png" alt="爬虫" title="爬虫" width="800" />

这样就完成了一个非常简单的自动更新Cookie的爬虫。这样就搞定了吗？显然还没有，正当我兴高采烈的写完这篇文章后，一看爬虫，又被封了。。而且这次连换Cookie也行不通了。看来新浪微博的反爬虫机制还要高级些，不仅通过Cookie，还会通过IP? 下一步如何应对，等我的IP解封后，再继续探索吧。。。( T T )

