<!--
.. title: 分布式全站爬取kuku漫画(scrapy-redis)
.. slug: fen-bu-shi-quan-zhan-pa-qu-kukuman-hua
.. date: 2016-12-21 05:42:31 UTC+08:00
.. tags: 
.. category: 爬虫
.. link: 
.. description: 
.. type: markdown
-->

## 思路
虽然scrapy支持多线程，但是单机scrapy也是有性能瓶颈的。使用scrapy-redis可以将scrapy改造成分布式的爬虫架构。

## 改造的原理如下
###### 相对于对于原版的scrapy，scrapy-redis修改了四个部分
 + 调度器(Scheduler)
 + 去重(Dupefilter)
 + 管道(Pipeline)
 + 抓取器(Spider)
 + 其核心是使用了redis代替了原本的queue
###### 主要思想是使用了redis代替了原本的queue
 + 因为redis可以作为一个消息队列
 + 这样多个爬虫实例就可以通过共享消息队列来实现分布式了。

## 要实现分布式主要要解决的问题：
 + 任务分发，让每个实例知道自己要做什么
 + 共享消息队列，不能出现重复爬取的情况
 + 这里Scheduler就是用来分发任务的
 + 如果配置了去重，Dupefilter会负责维护一个集合，包含抓取过的url，避免重复抓取
 + Pipeline可以是自己写的，也可以使用RedisPipeline将结果存储在redis中
 + 最后Spider中需要实现具体爬虫功能的部分，也是写代码的地方

## 正常爬取过程中，在Redis中会出现三个序列：
###### Redis中的key都是按照spidername:word的形式来命名的
 + 其中Dupefiler用来去重，需要合适的配置项
 + Requests包含了所有爬取的序列，是所有各个爬虫节点的任务来源
 + 同时所有在爬取过程中yield出的新request都会被放入这个序列
 + 最后如果你在配置项中使用了RedisPipeline，Spider返回的结果会被存储在Items中
 + 除了这三个序列，有时候会有一个名为start_urls序列

## 可如下部署这个分布式爬虫
 + scrapy本身支持并发和异步IO，通过修改其配置项CONCURRENT_REQUESTS来实现，默认为16
 + 因为GIL的存在，如果配置合理，此时的爬虫会占满你一个CPU核心
 + 通过手动执行N个爬虫实例（scrapy crawl comicCrawler），N一般是你的CPU核心数，你可以充分发挥出一台机器的性能
 + 同样，在其他机器上，你可以执行相同的操作，所有的爬虫实例都共享一个redis数据库

## 从scrapy配置改为scrapy-redis
###### 修改Setting.py
```
# SCHEDULER是核心部分，scrapy-redis重写了调度器用于管理分布式架构
SCHEDULER = "scrapy_redis.scheduler.Scheduler"
# DUPEFILTER_CLASS用用于去重
DUPEFILTER_CLASS = "scrapy_redis.dupefilter.RFPDupeFilter"
# SCHEDULER_PERSIST允许配置是否可以暂停\继续爬虫，互有利弊
SCHEDULER_PERSIST = True
# SCHEDULER_QUEUE_CLASS是队列的类型，有三种：PriorityQueue（默认）, FifoQueue和LifoQueue。
SCHEDULER_QUEUE_CLASS = PriorityQueue
LOG_LEVEL = 'DEBUG'

# Redis可以url或python字典的方式配置
# 通过url配置redis
# redis://YOUR_REDIS_HOST_IP:6379
# 通过python字典配置redis
REDIS_HOST = "YOUR_REDIS_HOST_IP"
REDIS_PORT = 6379
REDIS_PARAMS = { # 默认为0
# 如果需要的话，可以通过REDIS_PARAMS字典对具体的某个参数做调整
# 比如说，redis数据库中第0个实例可以改为其他数字(一般1到15)的第11个实例，避免与其他爬虫冲突。
    'db': 0
}
```
 + 其他的一般保持默认即可。
 + redis的优点是可以运行在内存中，存取速度快，用来存储到的数据太浪费了，所以按实际需求写pipeline就行
###### 修改spider文件
 + 所有涉及到的spider文件都需要修改，scarpy的spider类一般继承于scrapy.Spider，这里要调整为scrapy_redis.spiders。然后，没了。
<!--比如说：-->
<!--在我的例子中，我没有使用scrapy的Rules部件，因此在多页面爬取的时候，我需要为每一类请求都指定好合适的callback。你可以通过配置好Rules，它可以通过匹配请求的URL，然后给每一类URL绑定相应的回调函数。-->
<!--如果你没有在代码中配置start_urls，你需要指定一个redis的key值用来之后向其中推送请求，redis_key='comicCrawler:start_urls'。-->
<!--代码和效果-->
<!--代码我放在GitHub上了，-->
## 运行
 + 进入scrapy根目录,运行 scrapy crawl SPIDER_NAME
 + 我的笔记本是四核mac的，然后再加一个云主机，我一共起了5个实例，每个实例的并发数是16。最后跑出来，大概一个半小时可以抓到22W结果。

