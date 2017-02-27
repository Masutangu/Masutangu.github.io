---
layout: post
date: 2017-02-06T08:55:19+08:00
title: engineering blog 
category: 后台
---

[<Client-side ranking to more efficiently show people stories in feed>](https://code.facebook.com/posts/663139850520576/client-side-ranking-to-more-efficiently-show-people-stories-in-feed/)

文章提出如何优化 feed 流的体验。以往 feed 流在服务端进行排序后发送给客户端，客户端直接按序展示。这种方式有个局限，如果网络很差，包含图片视频资源的 feed 还没加载成功，客户端上展示的就是灰色方块。

根据以下两点原则：

* 用户无需依赖良好的网络环境才能流畅使用 facebook app ，弱网络条件下也应该有良好的用户体验。
* feed 只有在相关资源（图片／视频／音频等）已经加载完成后才有价值。

之前的架构图如下：

<img src="/assets/images/engineering-blog-note/illustration-1.png" width="800" />

优化后的架构如下：

<img src="/assets/images/engineering-blog-note/illustration-2.png" width="800" />

新的方案创建了一个 story pool，其作用在于将从 server 拉回来的 feed，和本地 persistent cache 中缓存的 feed 做排序，排序依据包括：

* server 侧的打分 
* feed 的类型 
* feed 包含的资源的加载情况 
* 用户网络环境

fb new feed 算法


[Instagram Search Architecture](https://engineering.instagram.com/search-architecture-eeb34a936d3a#.f2nqsaiq2)

* Instagram’s search infrastructure consists of a **denormalized** store of all entities of interest: hashtags, locations, users and media. In typical search literature these are called documents. Documents are grouped together into sets which can be queried using extremely efficient set operations such as AND, OR and NOT. 

* Instagram serves millions of requests per second. Many of these, such as signups, likes, and uploads, modify existing records and append new rows to our master PostgreSQL databases. To maintain the correct set of searchable documents, our search infrastructure needs to be notified of these changes. Furthermore, search typically needs more information than a single row in PostgreSQL.

To solve the problem of denormalization, we introduced a system called Slipstream where events on Instagram are encoded into a large Thrift structure containing more information than typical consumers would use. 

These events are binary-serialized and sent over an asynchronous pub/sub channel we call the Firehose. Consumers, such as search, subscribe to the Firehose, filter out irrelevant events and react to remaining events.

架构图：

<img src="/assets/images/engineering-blog-note/illustration-3.png" width="800" />

Since Thrift is schematized, we re-use objects across requests and have consumers consume messages without the need for custom deserializers. A subset of our Slipstream schema, corresponding to a photo like is shown below:

```
struct User {
1: required i64 id;
2: string username;
3: string fullname;
4: bool is_private;
...
}
struct Media {
1: required i64 id; 
2: required i64 owner_id;
3: required MediaContentType content_type;
...
}
struct LikeEvent {
1: required i64 liker_id;
2: required i64 media_id;
3: required i64 media_owner_id;
4: Media media;
5: User liker;
6: User media_owner;
...
8: bool is_following_media_owner;
}
union InstagramEvent {
...
2: LikeEvent like;
...
}
struct FirehoseEvent {
1: required i64 server_time_millis;
2: required InstagramEvent event;
    }
```

Firehose messages are treated as best-effort and a small percentage of data loss is expected in messaging. **We establish eventual consistency in search by a process of reconciliation or a base build.** Each night, we scrape a snapshot of all Instagram PostgreSQL databases to Hive for data archiving. Periodically, we query these Hive tables and construct all appropriate documents for each search vertical. The base build is merged against data derived from Slipstream to allow our systems to be eventually consistent even in the event of data loss.

Behind the scenes, queries to Unicorn are rewritten into S-Expressions that express clear intent, for example:

```
(and user:maxime (apply followed_by: followed_by:me))
```

which translates to “people named maxime followed by people I follow”. Our search infrastructure proceeds in two (intermixed) steps:

* Candidate generation: finding a set of documents that match a given query. Our backend dives into a structure called a reverse index, which finds sets of document ids indexed by a term. For example, we may find the set of users with the name “justin” in the “name:justin” term.
* Ranking: choosing the best documents from all the candidates. After getting candidate documents, we look up features which encode metadata about a document. 

The result of the two steps is an ordered list of the best documents for a given query.

As part of our search improvements, Instagram now takes into account who you follow and who they follow in order to provide a more personalized set of results. This means that it is easier for you to find someone based on the people you follow.

After candidate generation, we go through a process of ranking which chooses the best media by assigning a score to each document. The scoring function consumes a list of features and outputs a score representing the “goodness” of a given document for our query:

* Visual: features that look at the visual content of the image itself. Concretely, we run each of Instagram’s photo through a deep neural net (DNN) image classifier in an attempt to categorize the content of the photo. Afterwards, we perform face detection in order to determine the number and size each of the faces in the photo.

* Post metadata: features that look at non-visual content of a given post. Many Instagram posts contain captions, location tags, hashtags and/or mentions which aid in determining search relevancy.

* Author: features that look at the person who made a given post. Some of the richest information about a post is determined by the person that made it

Our media search infrastructure also extends itself into discovery, where we serve interesting content that users aren’t explicitly looking for.  Concretely, one source of explore candidates “photos liked by people whose photos you have liked”. We can can encode this into a single unicorn query with:

```
(apply liker:(extract owner: liker:<userid>))
```

This proceeds inwards-outwards by:

```
liker:<userid>: posts that you’ve liked
(extract owner:…): the owner of those posts
(apply liker:..): media liked by those owners
```

After this query generates candidates, we are able to leverage our existing ranking infrastructure to determine the top posts for you. Unlike top posts on hashtag and location pages, the scoring function for explore is machine-learned instead of hand tuned.


