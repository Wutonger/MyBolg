title: 一文带你轻松入门ElasticSearch

cover: /img/post/31.jpg

author: Loux
tags:

  - ElasticSearch
categories:
  - 后端知识
date: 2020-01-15 09:20:00

---

在日常开发中，对于Mysql等关系型数据库我们使用的总是比较多的。但是最近这些年由于互联网的爆炸式增长，在某一些场景中，使用关系型数据库会导致严重的性能问题。

例如一个博客网站，有几十万条博客文章，这个时候在首页需要有一个搜索功能，可以通过关键词对文章标题或者内容进行查询。若我们使用Mysql实现需求时，会发现搜索的时间可能需要几秒钟时间，严重影响到用户体验。

那么这是什么原因导致的呢？

既然我们要使用关键词去匹配，肯定需要用like查询，熟悉SQL语句的都知道，使用like可能会使索引失效，导致全表扫描，这样效率肯定就低了。

同时我们的数据库并不能分词，什么是分词呢？例如用户搜索了《ElasticSearch入门教程》，在查询时会根据输入的关键词进行拆分，例如上述的就可以拆分为“ElasticSearch”，“入门”，“教程”这三个词汇，进行查询，这样的好处是使得搜索到的结果更全面。可能文章库中并没有《ElasticSearch入门教程》这篇文章，但是有《ElasticSearch从入门到实践》这篇文章，这样分词搜索就会搜索到这篇文章，用户可以进行学习。若没有分词呢？用户搜索后发现一片空白，这样对用户体验是很不好的。

若我们自己实现分词算法，那么对于时间、精力来说耗费是巨大的，可能时间复杂度也会很高，导致搜索的时间变得更长。

对于搜索功能来说，使用关系型数据库的痛点实在太多太多了。还有例如根据关键词匹配的程度进行排序，匹配程度越高的排在越前面等等。

由上面分析，我们可以知道，在用户活跃数量多的网站，使用Mysql做全文搜索有至少三个痛点

* 效率不高
* 很难进行分词查询
* 不好实现结果集根据关键词的匹配程度进行排序

还好，在我们前面已经有了大佬比我们早一步想到了这一点。当我们使用ElasticSearch做为检索服务器，可以很好的解决以上三个痛点。

# ElasticSearch简介

> ElasticSearch是一个基于[Lucene](https://baike.baidu.com/item/Lucene/6753302)的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java语言开发的，并作为Apache许可条款下的开放源码发布，是一种流行的企业级搜索引擎。ElasticSearch用于[云计算](https://baike.baidu.com/item/云计算/9969353)中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。官方客户端在Java、.NET（C#）、PHP、Python、Apache Groovy、Ruby和许多其他语言中都是可用的。根据DB-Engines的排名显示，Elasticsearch是最受欢迎的企业搜索引擎，其次是Apache Solr，也是基于Lucene。            —来自百度百科

有以上可以知道，ES(下面都用ES代替ElasticSearch)在世界范围内都是很受欢迎的。同时它具有很高的**实时性**，也很容易搭建集群。最重要的一点来说，它是java语言开发的，同时公布了源代码。对于我们Java程序员来说是非常友好的。

那么目前有哪些知名网站在使用ElasticSearch呢？

* 2013年初，GitHub抛弃了Solr，采取ElasticSearch 来做PB级的搜索。 “GitHub使用ElasticSearch搜索20TB 的数据，包括13亿文件和1300亿行代码”
* 维基百科使用的搜索引擎是基于ES搭建的
* 百度目前广泛使用ElasticSearch作为文本数据分析，采集百度所有服务器上的各类指标数据及用户自 定义数据，通过对各种数据进行多维分析展示，辅助定位分析实例异常或业务层面异常

我们看到目前这些有名的大公司都在使用ES，作为一名程序员，我们必须要牢牢的掌握住ES的使用。关于ES的安装这里就不详细说了。

# ElasticSearch中的相关概念

ES中有一些概念，在我们使用它之前，必须要对其有一定的认识，不然学习时可能会云里雾里。我们可以对比Mysql来对Es中的一些属性进行分析：

| Mysql    | ElasticSearch |
| -------- | ------------- |
| Database | index         |
| table    | type          |
| row      | document      |
| column   | field         |
| 结构     | mapping       |

需要值得注意的是，在ES6中推荐一个index中只能有一个type，而在ES7中已经删除了type这个概念。

当然以上的类比也不是准确的，但是能让我们更快的入门ES。

# ES的安装

ES的安装十分简单，windows环境下直接在[官网](https://www.elastic.co/cn/)下载ElasticSearch的zip包解压缩即可。这里我使用的版本为**6.3.2**。在解压后启动bin目录下的elasticsearch.bat批处理文件，稍后打开浏览器输入localhost:9200,若显示以下界面，则说明ES的安装成功

![](/images/image-20200116112437950.png)

**需要注意，一定要提前安装好JDK8+以上的版本，并配置好环境变量。因为ES是基于java开发的，需要JAVA的运行环境。**

# ES中的语法

前面简介中提到了，ES中提供Restful Api来进行数据的访问。具体格式如下：

```bash
curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'
```



| 参数           | 解释                                                         |
| -------------- | ------------------------------------------------------------ |
| `VERB`         | 适当的 HTTP *方法* 或 *谓词* : `GET`、 `POST`、 `PUT`、 `HEAD` 或者 `DELETE`。 |
| `PROTOCOL`     | `http` 或者 `https`（如果你在 Elasticsearch 前面有一个 `https` 代理） |
| `HOST`         | Elasticsearch 集群中任意节点的主机名，或者用 `localhost` 代表本地机器上的节点。 |
| `PORT`         | 运行 Elasticsearch HTTP 服务的端口号，默认是 `9200` 。       |
| `PATH`         | API 的终端路径（例如 `_count` 将返回集群中文档数量）。Path 可能包含多个组件，例如：`_cluster/stats` 和 `_nodes/stats/jvm` 。 |
| `QUERY_STRING` | 任意可选的查询字符串参数 (例如 `?pretty` 将格式化地输出 JSON 返回值，使其更容易阅读) |
| `BODY`         | 一个 JSON 格式的请求体 (如果请求需要的话)                    |

例如我现在要创建一个索引名称叫myblog，用于存放我们的博客文章。语法如下：

>PUT      http://127.0.0.1:9200/myblog

我们可以使用PostMan来进行这个操作，但是推荐使用ES官方的可视化管理插件Kibana。

在ES的官网可以看到：

![](/images/image-20200115152754979.png)

官方都是推荐我们使用Kibana来对ES进行管理、监控等。

kibana的下载也很简单，但是要注意选择的版本最好和你的ES版本一致。比我我的ES版本为6.3.2，那么我下载kibana的版本也为6.3.2。下载后直接解压到任意目录就行了。运行bin目录中的`kibana.bat`文件即可运行。成功启动后在浏览器输入localhost:5601,即可看到以下界面

![image-20200115153221403](/images/image-20200115153221403.png)

相当简洁漂亮的界面，从右上角chrome浏览器插件可以看出，界面是使用React.js开发的。

在你的ES启动后，kibana会自动检测到并管理。下面我们可以使用kibana的Dev Tools控制台来进行一些Restful风格的Api请求。在kibana中就不用写ip地址加port那些东西了。例如我们想要添加一个名为test的index，可以这样写：

```bash
PUT test
```

在Dev Tools中执行这个命令：

![](/images/image-20200115153918476.png)

根据右边的相应信息，我们应该可以知道，请求成功了。创建了一个名为test的index。

## index和mapping的设置

我们在创建index的时候就可以设置mapping了,比如定义一些字段等等，例如

```bash
request url  

PUT     myBlog

request Body

{
    "mappings": {
        "article": {
            "properties": {
                "title": {
                	"type": "text",
                    "store": true,
                    "index":"analyzed",
                    "analyzer":"standard"
                },
                "content": {
                	"type": "text",
                    "store": true,
                    "index":"analyzed",
                    "analyzer":"standard"
                }
            }
        }
    }
}
```

在kibana的控制台发送请求：

![](/images/image-20200115154904247.png)

我们发现请求成功了。这个请求的含义就是创建了一个名为myblog的index,同时设置它的mapping—在index中创建了一个type名为article,并创建了两个字段，"title"和"content"，并且这两个字段的类型都为text。这应该很好理解。

同时我们还可以在创建完索引之后再设置mapping，就以刚刚的test为例



![](/images/image-20200115155819704.png)

可以看到，也是可以正确设置mapping的，值得注意的是Url中的type为article，必须要和request body中的type  article一致，不然则会报错。

我们要删除一个index也很简单，直接用Delete请求就行了，例如：

```bash
DELETE test
```

## documents的管理

我们已经创建好了index和type，也设置好mapping了，现在可以添加数据(documents)了。

首先，**在ES中会有一个默认字段为_id，在添加documents时我们可以指定 _id,否则ES会为我们生成一条默认的 _id**

添加文档的命令为：

```bash
POST myblog/article/1
```

requestBody的内容为：

```bash
{
	"title":"ElasticSearch是一个基于Lucene的搜索服务器",
	"content":"它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java开发的，并作为Apache许可条款下的开放源码发布，是当前流行的企业级搜索引擎。设计用于云计算中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。"
}
```

这里我们指定了文档的 _id为1,在kibana的Dev Tools中执行这条命令后，可以看到如下返回值：

![](/images/image-20200115170738731.png)

根据返回的信息来看，我们执行的操作成功了。

接下来我们可以看到这一条信息：

![](/images/image-20200115171857919.png)

文档的修改操作其实就是覆盖，也就是说再指定这个_id的值来进行一次POST请求，同时更改RequestBody中的内容，这样这条数据就被更改了。

删除操作的命令为：

```bash
DELETE myblog/article/1
```

根据_id查询文档的命令为：

```bash
GET myblog/article/1
```

## queryString与term查询

当然根据_id来查询太简单了。我们还需要一些复杂的查询。例如根据关键词查询，根据String查询。

**term查询：**

url为：

`POST myblog/article/_search`

requestBody为：

```bash
{
  "query": {
    "term": {
      "title":"详"
    }
  }
}
```

我们以title中的关键字关键字“详”进行查询。

但是我们发现使用term查询只能输入一个汉字，若输入了多个汉字，查询到的结果就会为null，这是因为在我们设置myblog的mapping时，title字段"index"默认为“analyzed”，也就是说title中的每个词都会被分词器拆分后存储在title的索引中。例如“你好世界”在title的索引中会被拆分为["你","好","世","界"]，由于term是完全匹配，当你用“你好”进行搜索时，当然匹配不到任何结果，只有用单个字去搜索，才会匹配到正确的结果。

**queryString查询：**

请求的url和term查询是一样的，唯一不同的则是请求体。

```bash
{
  "query": {
    "query_string": {
      "default_field": "title",
      "query": "钢索"
    }
  }
}
```

![](/images/image-20200116103334852.png)

我们会发现搜索“钢索”为什么带“索”字的文档也在结果集中呢？这是因为ES中存在分词器这个概念，"钢索"这个短语会被ES中的标准分词器分为“刚”和“索”，这样带“刚”和“索”的title就都会被搜索到。

