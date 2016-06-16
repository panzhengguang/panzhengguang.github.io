---
layout: post
title: 信息检索之简单搜索引擎的实现
categories: IR
---
## 一、题目要求：

新闻搜索：定向采集3-4个体育新闻网站，实现这些网站信息的抽取、索引和检索。网页数目不少于10万条。能按相关度、时间、热度(需要自己定义)等属性进行排序，能实现相似新闻的自动聚类。

## 二、题目分析

题目分析：我们将任务分解为四个部分：新闻数据的爬取、倒排索引的构建、向量空间模型的实现 和 前端界面。

主要分为四个模块：网络爬虫、构建索引、文档评分、排序显示。其中模块与模块之间又包含一些子模块，包括：网页信息抽取、数据存储、文本分析、tf-idf 权重计算、向量空间模型建模、相关度热度时间排序、相似文档聚类、相关搜索推荐等。下面是整个搜索引起的设计结构图：


> [![image](http://images.cnitblog.com/blog/442949/201501/020950397943276.png "image")](http://images.cnitblog.com/blog/442949/201501/020950385591519.png)

## 三、网络爬虫

在本次作业中，网络爬虫部分使用了 Scrapy 开源框架。

Scrapy 是 Python 开发的一个快速,高层次的屏幕抓取和 web 抓取框架，用于抓取 web 站点并从页面中提取结构化的数据。

本模块的流程图：

> [![image](http://images.cnitblog.com/blog/442949/201501/020950407472118.png "image")](http://images.cnitblog.com/blog/442949/201501/020950401538232.png) 

### 3.1定义抽取数据的格式

对于爬虫的设计，首先需要确定抽取数据的格式，我们通过需求分析，决定抽取以下10 种数据。具体介绍如下：

* Artical：新闻的正文，用于文本分析和索引构建；
* ID：新闻的唯一标示符，在抓取过程中用于防止重复抓取某一新闻页面；
* Keyword：新闻中的关键字，用于相关搜索推荐；
* Show：新闻的评论数，用于热度的定义，方便按热度进行排序；
* Reply：新闻评论的回复数，用于热度的定义，方便按热度进行排序；
* Total：新闻的总共参与人数，用于热度的定义，方便按热度进行排序；
* Source：新闻的来源，用于后续扩展，区分 sina，sohu，163 等；
* Time：新闻的发布时间，用于后续的按时间排序
* Title：新闻的标题；
* URL：新闻的链接，用于 web 显示；

### 3.2新闻页面解析

在确定数据格式后，接下来要抓取某一个体育新闻页面， 然后解析出与上述数据格式相对应的数据。我们通过引入了一个HTML解析包HtmlXPathSelector来对页面信息进行提取。这里简单举一个例子进行说明：

[![image](http://images.cnitblog.com/blog/442949/201501/020950437474602.png "image")](http://images.cnitblog.com/blog/442949/201501/020950413885288.png)

## 四、倒排索引的构建

倒排索引是信息检索的重要一环。在这个模块中，主要包含四个关键步骤：从 json 文件中提取新闻内容、新闻内容切词、统计词频和文档频率、计算 tf-idf 权重。

[![image](http://images.cnitblog.com/blog/442949/201501/020950456388056.png "image")](http://images.cnitblog.com/blog/442949/201501/020950448419401.png)
### 4.1 文档切词

在常见的中文分词工具中，IKAnalyzer 分词器常用，效果好，方便拓展，受到了很多好评。因此，在我们的方案中，选用了 IKAnalyzer 分词器。IKAnalyzer 是一个开源的，基于 java 语言开发的轻量级的中文分词工具包。从 2006年 12 月推出 1.0 版开始， IKAnalyzer 已经推出了 4 个大版本。

分词效果示例：

IK Analyzer 2012 版本支持 细粒度切分 和 智能切分，以下是两种切分方式的演示样例。

> 文本原文 1:
> IKAnalyzer 是一个开源的，基于 java 语言开发的轻量级的中文分词工具包。从 2006年 12 月推出 1.0 版开始， IKAnalyzer 已经推出了 3 个大版本。

> 智能分词结果:
> ikanalyzer |是|一个|开源|的|基于|java|语言| 开发 | 的 | 轻量级 | 的 | 中文 | 分词 | 工具包 | 从 | 2006年 | 12月 | 推出 | 1.0版 | 开始 | ikanalyzer | 已经 | 推| 出了 | 3个 | 大 | 版本

> 最细粒度分词结果:
> ikanalyzer | 是 | 一个 | 一 | 个 | 开源 | 的 | 基于 | java | 语言 | 开发 | 的 | 轻量级|的 | 中文 | 分词 | 工具包 | 工具 | 包 | 从 | 2006 | 年 | 12 | 月 | 推出 | 1.0 |版 | 开始 |ikanalyzer | 已经 | 推出 | 出了 | 3 | 个 | 大 | 版本。

安装部署方法：它 的 安 装 部 署 十 分 简 单 ， 将 IKAnalyzer2012.jar 部 署 于项目的 lib 目录中 ；IKAnalyzer.cfg.xml 与 stopword.dic 文件放置 class 根目录（对于 web项目，通常是 WEB-INF/classes 目录，同 hibernate、log4j 等配置文件相同）下即可。

主要流程：将上一步提取的新闻主体的 txt 文档，全部遍历切词，将切词结果保存中本地中。

### 4.2倒排索引构建流程

在本题目中，要求是 10W 篇文档，数据量不是太大。因此，我们采用基于内存的索引构建方式，将索引构建的过程全部放入内存中进行统计，单次扫描，单次统计出结果，构建出索引。

主要流程：将上一步中分词的 10W 篇文档，全部遍历。在循环中，我们统计文档中每个词项出现的位置和次数，以及在 10W 篇文档中的总次数。在具体实现中，我们用两个HashMap 来保存结果，HashMap 用词项做键，文档名、文档位置及其他属性为值（中间用特殊符号分割开），一个用于统计每篇文档的情况，一个统计所有文档的情况。

[![image](http://images.cnitblog.com/blog/442949/201501/020950497945554.png "image")](http://images.cnitblog.com/blog/442949/201501/020950479665613.png)

### 4.3 计算td-idf权重

信息检索领域最出名的权重计算方法，tf-idf 权重计算公式：

[![image](http://images.cnitblog.com/blog/442949/201501/020950505443926.png "image")](http://images.cnitblog.com/blog/442949/201501/020950501843268.png) 

其中，dft 是出现词项 t 的文档数目成为逆文档频率，tft,d 是指 t 在 d 中出现的次数，是与文档相关的一个量，成为词项频率。随着词项频率的增大而增大，随着词项罕见度的增加而增大。

[![image](http://images.cnitblog.com/blog/442949/201501/020950532163210.png "image")](http://images.cnitblog.com/blog/442949/201501/020950519662925.png)

## 五、评分排序

采用向量空间模型来计算文档与查询的相似度，并进行排序，我们将查询和文档都表示为词项空间中的向量，进而可以计算这两个向量之间的相似度。计算相似度时，我们采用余弦相似度进行计算。

[![](http://images.cnitblog.com/blog/442949/201501/020950537165123.png)](http://images.cnitblog.com/blog/442949/201501/020950537165123.png)

文档向量化方法：利用tf-idf权重进行表示

计算相似度公式：

[![image](http://images.cnitblog.com/blog/442949/201501/020950547471708.png "image")](http://images.cnitblog.com/blog/442949/201501/020950540448322.png)

排序方法：根据相似度计算结果，按照得分高低来进行排序。 同时，可以根据时间和热度进行简单排序。 

## 六、前端界面

界面简答明了，风格简洁。包括一个用户输入界面和一个输出界面。

效果截图：

[![image](http://images.cnitblog.com/blog/442949/201501/020951037167401.png "image")](http://images.cnitblog.com/blog/442949/201501/020950594198647.png)

目前这个摘要生成技术还很丑。

note:本代码仅供参考，大家主要是把整个流程搞清楚就行，这程序依赖一些环境，各位的机器不一定能跑起来。

---------------------------------------------------------------------------------------

**如果各位觉得这篇博客和代码对您有一定帮助，还请您给我的下面的github地址一颗星，谢谢各位。**

**地址：[Simple_Search_Engine](https://github.com/panzhengguang/Simple_Search_Engine)**

------------

作者：[panzhengguang](https://github.com/panzhengguang)

记录：6-15日迁移自我的博客园[西芒xiaoP](http://www.cnblogs.com/panweishadow/)

若用于非商业目的，您可以自由转载，但请保留原作者信息和文章链接URL。