---
layout: post
date: 2016-01-26T12:42:43+08:00
title: 水滴石穿－第二期
category: 读书笔记
---

> 不积跬步，无以至千里；不积小流，无以成江海

## [A Little Architecture](http://blog.cleancoder.com/uncle-bob/2016/01/04/ALittleArchitecture.html)

文章以问答的形式，讲解了如何成为一名优秀的架构师。

成为一名优秀的架构师，不仅仅是知道如何选择数据库，选择框架。**一个优秀的架构，业务逻辑不需要依赖底层的技术架构。**

文章提出两个重要的原则：

* **Dependency Inversion Principle**
	> The source code of the sender does not mention, or depend upon, the source code of the receiver. In fact the source code of the receiver depends upon the source code of the sender.

	>The principles of architecture, of course. Senders own the interfaces that the receivers must implement.

* **Interface Segregation Principle**

	> Each business rule class will only use some of the facilities of the database. And so each business rule provides an interface that gives it access to just those facilities.

	良好的接口设计，能够让你延后做出技术选型的决定。你可以在初期使用轻量级的技术框架来实现功能，后期再根据实际情况进行调整。而你出色的接口设计使得你的业务逻辑不会依赖底层技术，换句话说，底层使用什么技术框架对于业务逻辑来说都是透明的。

	> Blessed is the team whose architects have provided the means by which all these decisions can be deferred until there is enough information to make them.

	> Blessed is the team whose architects have so isolated them from slow and resource hungry IO devices and frameworks that they can create fast and lightweight test environments.

	> Blessed is the team whose architects care about what really matters, and defer those things that don’t.

## [Why’d You Do That?!? An Engineer’s Guide to Debugging User Behavior](http://www.theeffectiveengineer.com/blog/debugging-user-behavior)

代码不符合我们的预期，我们可以通过各种方法进行调试，比如打断点，打log等。那如果用户的行为不符合我们的预期，我们应该如何debug用户呢？

文章提了几种方法来debug用户：

* **A/B test**

* **session logs**

* **user tests**:
	Let engaged customers or early adopters beta test a feature and then collect their feedback in a doc or talk to them in person.

## [How to be awesome Swift developer](http://blog.krzyzanowskim.com/2015/12/28/how-to-be-awesome-swift-developer/) 

如何成为了不起的swift程序员？作者给了一些很有用的建议，摘抄如下：

* embrace legacy
	do:
	* experiment a lot 
	* don’t be afraid doing things in non optimal way — wrong is way better than none
	* open your mind, try new things
	* read blogs of other developers
	* learn by doing
	* check what’s inside to understand it more
* don’t be a douchebag
	don’t:
	* my code is better than yours
	* don’t complain to much about what’s done (your work or in general)
	* for God’s sake stop telling people that tools you’re using are the best in the world just because you know how to use it
	* programming language doesn’t matter.
	* avoid “I know better” attitude
	* don’t be a douchebag

## [Introduction to MVVM Introduction to MVVM](https://www.objc.io/issues/13-architecture/mvvm/)

MVC有时被戏称为Massive View Controller，因为Controller的逻辑太重了。而且大部分是展示逻辑：把Model的数据转化为View的展现形式。

MVVM在MVC的基础上，将Controller的展示逻辑抽离出一个View Model层，这样不仅减轻了Controller的逻辑，简化了代码。同时也易于测试。

延伸阅读：

[Model-View-ViewModel for iOS Model-View-ViewModel for iOS](http://www.teehanlax.com/blog/model-view-viewmodel-for-ios/)

## [The Ultimate Guide to Minimum Viable Products](http://scalemybusiness.com/the-ultimate-guide-to-minimum-viable-products/)

最简可行产品(MVP)是指以最低成本尽可能展现核心概念的产品策略，即是指用最快、最简明的方式建立一个可用的产品原型，这个原型要表达出你产品最终想要的效果，然后通过迭代来完善细节。

> **A minimum viable product is therefore not a product. It is a minimum viable go to market step.**

这篇文章列举了7个MVP例子：

* Explainer Video
	以dropbox为例，用一段视频来介绍你的产品，从而收集用户反馈。

* A Landing Page
	善用登陆页来向用户展示核心信息，并借助统计工具（例如Google Analytics）来分析转化率／用户行为。

* Wizard of Oz MVP
	Put up a front that looks like a real working product, but you manually carry out product functions.
	以Zappos为例，初期是创始人到实体鞋店把照片拍下来，放到网店上，用户下单后再去实体店买回来并邮寄给用户这种方式运作的。通过这种方式，几乎不需要什么成本就可以清楚用户是否有网上购鞋的需求。当需求明确之后，再开始开发整个网站，从手工转向自动化。

* Concierge MVP
	和Wizard of Oz MVP类似，前期人工提供服务，逐渐改进，直到后期用户逐渐增多，再慢慢使用软件来代替人工。

* Piecemeal MVP
	和Concierge MVP及Wizard of Oz MVP类似，但是借助现有工具而不是manually去完成整个流程。

* Raise Funds from Customers
	众筹，不仅能验证你的产品是否有市场，还能拿到资金投入到产品之中。

* A Single Featured MVP
	Chances are that if you cannot find that one killer feature that can stand on its own — at least in with early users — adding more features will not make the product a must have.
	专注在简单，功能单一的功能。

最后文章的总结：

> The 20 second summary of this lesson is: don’t burn your money on a product no one will want to use. Get creative and think hard about what is the minimum thing you can do now to make sure that doesn’t happen:
* Select one MVP strategy you think would work for you
* Create a simple plan to execute on it (remember the “minimal” in MVP)

## [Meta-analysis: best interview questions to spot ideal employees](http://www.williambharding.com/blog/hiring/meta-analysis-best-interview-questions-to-spot-ideal-employees/)

招聘者最看重的特质是什么？

有团队意识，适应性强，值得信赖，价值观相符。

另外后面还有些经典的面试题，供面试者和面试官参考学习。

## [Square Defangs Difficult Decisions with this System — Here’s How](http://firstround.com/review/square-defangs-difficult-decisions-with-this-system-heres-how/?ct=t%28How_Does_Your_Leadership_Team_Rate_12_3_2015%29)

文章介绍SPADE，如何运用它来更好地做决策。

* Setting: The setting has three parts: what, when and why
	* Precisely define the decision to capture the “what.”
	* Calendar the exact timeline for the decision to realize the “when.”
	* Parse the objective from the plan to isolate the “why.”

* People: This includes those who are consulted and give input towards the decision, the person whoapproves the decision, and most importantly the person who’sresponsible for ultimately making the call
	* Synonymize accountability and responsibility. 亲身经验告诉我，这点非常重要！
	> “At Square, the person who’s responsible for making the decision is the person who’s accountable for its execution and success.
	
	* Veto decisions mainly for their quality, not necessarily their result.
	* Formally recognize the roles of all active participants.

	> Listening matters. Much, much more than you think. People want the option to chime or chip in, even if their stance is counter to the end decision. They just want to be listened to.

* Alternatives: Alternatives should be feasible — they should be realistic; diverse — they should not all be micro-variants of the same situation; and comprehensive — they should cover the problem space

	>“Get in a room, get on a whiteboard, and brainstorm,” says Rajaram. “For each alternative, list out the pros and cons, as well as the parameters behind the quantitative model. There are no shortcuts. Get into the numbers as much as possible. It can be very hard with ambiguous decisions to get down into the numbers, but it’s very valuable to do so.”

	数字更直观，尽量把各个方案转化为数值进行比较。

* Decide
	* The most important part part of this process is to ask them to send you their vote privately.（也许匿名是个更好的主意）
	* That said, in exceptional circumstances, choices can be articulated openly.

* Explain
	* Run your decision and the process by the Approver.
	* Convene a commitment meeting.
	* Circulate the annals of the decision for precedent and posterity.

## [Full text search in milliseconds with PostgreSQL ](https://blog.lateral.io/2015/05/full-text-search-in-milliseconds-with-postgresql/)

简单介绍了如何用PostgreSQL做全文检索。

## [Why I Quit my Dream Job at Ubisoft Why I Quit my Dream Job at Ubisoft](http://gingearstudio.com/why-i-quit-my-dream-job-at-ubisoft)

作者讲述了自己从大公司辞职，自己组团队做独立游戏的原因：

* 每个人的贡献被稀释了，因此缺少了动力。
* 沟通成本变高了。

> When people realize they’re just one very replaceable person on a massive production chain, you can imagine it impacts their motivation.  
> No matter what’s your job, you don’t have a significant contribution on the game. You’re a drop in a glass of water, and as soon as you realize it, your ownership will evaporate in the sun. And without ownership, no motivation.

## [消息系统设计与实现 稀土掘金：消息系统设计与实现「上篇」 ](http://gold.xitu.io/entry/5649618800b0ee7f5991a717)

分析了消息系统的设计与实现，非常有参考价值。

## [张小龙谈微信价值观 微信公开课PRO版] (http://v.qq.com/cover/a/a7v5hfc9umds0c9.html?vid=s0019dietdc)

微信的四个价值观：

* 一切以用户价值为依归。强调公平公正，善良比聪明更重要。
* 让创造发挥价值。
* 用完即走，提高用户效率。帮用户过滤掉无用的信息，只留下用户关心的价值点。做法：
	* 限制营销信息，简化朋友圈内容 
	*  提高加好友门槛 
	*  限制公众号下发消息的频率。

	如果用户在你产品上找不到价值，或者很难找到价值（无用信息太多，骚扰太多），就会离开你的产品。

* 让商业化存在于无形之中。

最后张小龙剧透了下微信的应用号，听起来很有趣，以后可以尝试一下。

## [Amazon founder and CEO Jeff Bezos delivers graduation speech at Princeton University](https://www.youtube.com/watch?v=vBmavNoChZc)

[原文链接](http://www.princeton.edu/main/news/archive/S27/52/51O99/index.xml)

“善良比聪明更重要，选择比天赋更重要”。最后的一系列反问句满满的正能量：

> How will you use your gifts? What choices will you make?

> Will inertia be your guide, or will you follow your passions?

> Will you follow dogma, or will you be original?

> Will you choose a life of ease, or a life of service and adventure?

> Will you wilt under criticism, or will you follow your convictions?

> Will you bluff it out when you’re wrong, or will you apologize?

> When it’s tough, will you give up, or will you be relentless?

> Will you be a cynic, or will you be a builder?

> Will you be clever at the expense of others, or will you be kind?

## 一勺鸡汤 
**How do you know you are on the right path in your life? When you no longer hate Monday mornings.**

