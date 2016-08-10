---
layout: post
title:  "Welcome to Jekyll!"
date:   2016-02-05 00:31:28 +0800
categories: jekyll
---
You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets:

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: http://jekyllrb.com/docs/home


# 攻略推荐流程简介

@(年度工作总结-2015年)[推荐, Storm, Spark, LR]

-------------------


[TOC]
## 业务简介
> 针对掌盟和王者荣耀手机助手等应用，提供个性化的攻略推荐，有助于优化用户体验。
##本文目的
>简要介绍攻略推荐处理流程，便于后期业务复用和细节优化。
>介绍攻略和用户属性的特征工程化处理
>介绍逻辑回归模型的使用


## 攻略特征生成

### 数据源准备

####数据源
一般各个业务方面的攻略存储有差异，例如有放在cdb中或者redis中。攻略推荐的原始数据主要需要下面几个字段：攻略Id、中文标题、作者、攻略内容、发布时间、播放次数等其他字段。攻略内容大都以原始的xml形式给出，例如：
```xml
<p><strong><strong style="white-space: normal;">&nbsp; &nbsp; &nbsp; &nbsp;——</strong>解析</strong></p><p>&nbsp; &nbsp; &nbsp; &nbsp;1.线上续航能力神技能，配合上出门装多兰盾，还有防御天赋的两点“愈合”，你就知道前期的德玛西亚之力盖伦有多恶心人了！<br/></p><p>&nbsp; &nbsp; &nbsp; &nbsp;2.如果对拼大亏后，果断舍弃补兵，躲草丛吃经验，一分钟后又是一个名扬天下的草丛伦！</p>
```
####如何从xml网页文档提取中文
```python
from bs4 import BeautifulSoup
```
这个库可以有效提取xml网页中的中文，在爬虫中比较常用。提取中文之后如下：
```text
——解析 1.线上续航能力神技能，配合上出门装多兰盾，还有防御天赋的两点“愈合”，你就知道前期的德玛西亚之力盖伦有多恶心人了！    2.如果对拼大亏后，果断舍弃补兵，躲草丛吃经验，一分钟后又是一个名扬天下的草丛伦！
```
####构建词库
分别为作者、标题、内容构建词库，为后期的dummy化特征作准备。
####构建英雄字典
这个在提取标题关键词和攻略文本分词时比较重要，例如在LOL中的英雄盖伦，盖伦是名字，德玛西亚是称号，草丛伦是外号，这个都应被判定为盖伦。
####分词与自定义字典
[Jieba](http://www.oschina.net/p/jieba)分词简单易用，效率高。在利用Jieba分词时，需要预先加载一个自定义分词字典。这个字段主要包括英雄的称号、名字、技能、符文、天赋以及常见的游戏解说名字等。例如如下自定义字典:
```text
德玛西亚
盖伦
草丛伦
多兰盾
```
在分词之前预加载自定义的字典，则可以保持“草丛伦”不会被分成“草丛”和“伦”。

####词性过滤
分词之后需要对词性进行标注并进行词性过滤。
词性标注如下：
```python
import jieba.posseg as pseg
words =pseg.cut("线上续航能力神技能，配合上出门装多兰盾，还有防御天赋的两点“愈合”，你就知道前期的德玛西亚之力盖伦有多恶心人了!如果对拼大亏后，果断舍弃补兵，躲草丛吃经验，一分钟后又是一个名扬天下的草丛伦！",HMM=True)
```
对于词性过滤一般要求如下：
```python
对分词结果，删除停用词、频繁无用词、单字词，只保留以下词性的词语：
（1）名词类：n（名词），nr（人名），ns（地名），nt（机构团体名），nz（其他专用名），ng（名词性词素）
（2）动词类：v（动词），vn（名动词），vl（动词性惯用语），vg（动词性语素）
（3）形容词类：a（形容词），an（名形词），ag（形容词性语素），al（形容词性惯用语）
（4）英文：eng（英文）
```
最后会剩下以下词语:
```text
续航 能力 技能 配合 出门 多兰盾 还有 防御 天赋 愈合 知道 德玛西亚 盖伦 恶心
舍弃 补兵 草丛 经验 草丛伦
```
实践过程中需要注意到unicode、utf8以及gbk等编码之间的转化

###  攻略关键词提取
经过上面的分词和词性过滤，每篇攻略将过滤得到比较重要的词语，接下来需要对几千篇攻略分析，应用$TF-IDF$来提取每篇攻略的关键词。
####$TF-IDF$关键词提取原理
$TF-IDF$($Term frequency-inverse document frequency$ ) 是文本挖掘中一种广泛使用的特征向量化方法。

假设单词用$t$表示，文档用$d$表示，语料用$D$表示，那么文档频度$DF(t, D)$是包含单词$t$的文档数。如果我们只是使用词频度量重要性，就会很容易过分强调重复次数多但携带信息少的单词，例如：“a”, “the”以及“of”。如果某个单词在整个语料库中高频出现，意味着它没有携带专门针对某特殊文档的信息。

其中词频$TF$指的是某一个给定的词语在该文件中出现的次数。$TF$通常要被归一化（区别于下面的$IDF$，分子小于分母）：

$$TF(t,d) = \frac{t}{d}$$

逆文档频度$IDF$是单词携带信息量的数值度量:

$$IDF(t,D) = \log \frac{{|D| + 1}}{{DF(t,D) + 1}}$$

其中$|D|$是语料中的文档总数。由于使用了$log$计算，如果单词在所有文档中出现，那么$IDF$就等于0。注意这里做了平滑处理（+1操作），防止单词没有在语料中出现时IDF计算中除0。$TF-IDF$度量是$TF$和$IDF$的简单相乘：

$$TFIDF(t,d,D) = TF(t,d) \cdot IDF(t,D)$$

####具体实现
```python
from  sklearn import feature_extraction
from  sklearn.feature_extraction.text import TfidfTransformer
from  sklearn.feature_extraction.text import CountVectorizer
vectorizer=CountVectorizer(min_df=0.005,max_df=0.6)
transformer=TfidfTransformer()
tfidf=transformer.fit_transform(vectorizer.fit_transform(Corpus))
word=vectorizer.get_feature_names()
```
经过$TF-IDF$之后，我们会为每一篇攻略内容保留至多10个关键词。关键词在词库索引中查找对应的编号，则每篇攻略就由不超过10个关键词的索引构成，例如某篇攻略关键词提取之后包含“多兰盾  防御  盖伦”，则根据词库可能会被编码成“237 896  145”。具体$TF-IDF$计算时需要注意设置合适的文档频DF阈值。
### 离散Dummy化
上面得到每篇攻略的关键词索引可以直接Dummy化。然而每篇攻略上线之后的统计数据，例如点击率、播放次数等特征则需要进行离散Dummy化，具体公式如下：
$$id=\dfrac{x_i-x_{min}}{x_{max}-x_{min}}*dummyRange$$

### 生成特征并写入Tcaplus
为了后面Storm实时拼接样本以及推荐系统生成用户特征提供高速存储。
	
	TCaplus是互娱研发部结合游戏特点、平衡性能和成本，开发的一款高速分布式Key-Values模型的NoSql存储系统。与之类似的是Redis，这是一个开源的使用ANSI C语言编写、支持网络、可基于内存亦可持久化的日志型、Key-Value数据库。

## 用户特征生成

###  用户自然人属性计算
构建用户自然人属性宽表 ，主要包括用户基本信息、登录、活跃、付费、最近常玩英雄以及实力表现等数据。用户自然人属性和攻略属性的差异在于，基本来自结构化的数据处理，不再需要通过分析xml内容去提取关键词特征。

### 离散Dummy化
将之前的宽表各个字段根据要求分别进行离散Dummy化。

### 生成特征并写入Tcaplus
类似攻略属性。


## Storm实时统计与样本拼接
###什么是Storm
 Storm是一个免费开源、分布式、高容错的实时计算系统。Storm令持续不断的流计算变得容易，弥补了Hadoop批处理所不能满足的实时要求。Storm经常用于在实时分析、在线机器学习、持续计算、分布式远程调用和ETL等领域。Storm的部署管理非常简单，而且，在同类的流式计算工具，Storm的性能也是非常出众的。
 
Storm主要分为两种组件Nimbus和Supervisor。这两种组件都是快速失败的，没有状态。任务状态和心跳信息等都保存在Zookeeper上的，提交的代码资源都在本地机器的硬盘上。

Storm的常见关键词简介如下：
	
>Nimbus负责在集群里面发送代码，分配工作给机器，并且监控状态。全局只有一个。
>Supervisor会监听分配给它那台机器的工作，根据需要启动/关闭工作进程Worker。每一个要运行Storm的机器上都要部署一个。
>Zookeeper是Storm重点依赖的外部资源。Nimbus和Supervisor甚至实际运行的Worker都是把心跳保存在Zookeeper上的。Nimbus也是根据Zookeerper上的心跳和任务运行状况，进行调度和任务分配的。
>Storm提交运行的程序称为Topology。
>Topology处理的最小的消息单位是一个Tuple，也就是一个任意对象的数组。
>Topology由Spout和Bolt构成。Spout是发出Tuple的结点。Bolt可以随意订阅某个Spout或者Bolt发出的Tuple。Spout和Bolt都统称为component。

###日志接入与样本拼接
![Alt text](./1469452773370.png)
一方面由于客户端上报给我们的日志可能有多种形式，例如Probuf、Json或者Tlog等形式，另一方面由于考虑到后面Storm的处理能力，例如节假日一般数倍于平时的日志量。我们通过几乎透明的中转把数据发送到Tdbank数据银行，它既可以起到后面缓冲消费的作用，并且可以将数据共享给其他合作的业务部门消费，起到了充分利用数据的目的。
接下来通过Storm来消费Tdbank 的数据，主要有以下几个目的：
1、PV统计：主要包括曝光总量、点击总量、免费领用总量、购买总量、购买流水
2、UV统计：主要包括曝光、点击、购买等对用户进行去重的数据
3、Filter：这个主要是用来在业务上线或者调试时，方便对关注的用户进行过滤，校验数据协议的准确性
4、Sample：这个是用来拼接模型训练样本，每条样本包括用户特征、广告特征以及交叉特征
5、Save：对部分数据保存至tdw，方便后续的深入分析
更加深入的可以参考我的同事文章[《图灵系统介绍（十）- 实时日志处理平台》](http://km.oa.com/group/25372/articles/show/232795)

####Storm实现方法
在execute方法中，传入的参数是一个Tuple，该Tuple就包含了上游（Upstream）组件ProduceRecordSpout所emit的数据，直接取出数据进行处理。上面代码中，我们将取出的数据，按照空格进行的split，得到一个一个的单词，然后在emit到下一个组件，声明的输出schema为2个Field：word和count，当然这里面count的值都为1。
```java
public static class WordSplitterBolt extends BaseRichBolt {
     private static final long serialVersionUID = 1L;
     private static final Log LOG = LogFactory.getLog(WordSplitterBolt.class);
     private OutputCollector collector;
    
     @Override
     public void prepare(Map stormConf, TopologyContext context,
               OutputCollector collector) {
          this.collector = collector;              
     }

     @Override
     public void execute(Tuple input) {
          String record = input.getString(0);
          if(record != null && !record.trim().isEmpty()) {
               for(String word : record.split("\\s+")) {
                    collector.emit(input, new Values(word, 1));
                    LOG.info("Emitted: word=" + word);
                    collector.ack(input);
               }
          }
     }

     @Override
     public void declareOutputFields(OutputFieldsDeclarer declarer) {
          declarer.declare(new Fields("word", "count"));         
     }
    
}
```
####广告pb定义

```java
package gpm;
option java_package = "gpm.logprocess.proto";
option java_outer_classname = "TrainingProto";

enum Constants {
    SPEED_MAX_ITEM_ID = 100000;
}

enum ValueType {
    DISCRETE = 0;
    DUMMY    = 1;
}

message IDValuePair
{
    required int64     id = 1;
    optional float     value = 2 [ default  = 1.0 ];
    optional ValueType value_type = 3 [default  =  DUMMY];
}

message Features
{
    repeated IDValuePair features = 1;
    repeated int64       feature_group_ids = 2; // 有feature值的group id集合
}

message TrainingSample
{
    repeated IDValuePair features  = 1;
    required float       y_value   = 2;
    required int32       timestamp = 3;
    optional int64       id        = 4;
}

message TrainingResult
{
    required float       theta_zero = 1;
    repeated IDValuePair thetas = 2;
}

```

###样本拼接
LR模型比较简单，需要交叉特征提升模型表达能力。
在前面已经将用户属性和攻略属性分别离散Dummy化存储至KV系统Tcaplus，在拼接样本的bolt中，对来自spout的每条点击流数据判断是曝光还是点击，如果是点击则标记为正样本，否则为负样本。接下来对该条数据的uin去缓存查有无该用户数据，没有则至Tcaplus中去查询用户数据，得到uin的特征之后则查询攻略的特征，也是优先从缓存去查，然后去Tcaplus查询特征，接下来根据高阶交叉规则拼接模型样本。
大部分情况下，待推荐的有效攻略不过几千，而用户量是远大于攻略数目的，例如掌盟的日均曝光UV可达500万，所以实际情况下，每条点击流数据主要是去Tcaplus去查询用户特征，而攻略特征在bolt运行一段时间之后大部分已经保存在Cache中。
这里还要注意一点，在机器学习的模型训练中，正负样本的比例控制很重要。点击流中的曝光，也就是负样本远比正样本多，所以对负样本要做一定的筛选，使正负样本比例在一个合理的阈值范围之内，保证模型训练的有效性。


##逻辑回归模型训练
在计算广告中，常使用逻辑回归模型，因为LR模型比较简单，易于大规模并行化。另外需要注意在特征工程中关注有效提取特征，特征的维度如果过高，后续的计算复杂度将会提高，需要考虑正则化。
###最大似然法ML
逻辑回归其实仅为在线性回归的基础上，套用了一个逻辑函数，但也就由于这个逻辑函数，逻辑回归成为了机器学习领域一颗耀眼的明星，更是计算广告学的核心。
\begin{align}
 & y=f(x)=w^Tx\tag{1} \\
  & y=f(x)=sign(w^Tx)\tag{2} \\
  & y=f(x)=\dfrac{1}{1+\exp(-w^Tx)} \tag{3}
\end{align}


 假设$x\in R^d$  为$d$维输入向量，$y\in \{0,1\}$为输出标签，$w\in R^d$是参数。
 则有
\begin{align}
 & p(y_i=1|x_i,w)=\dfrac{\exp(w^Tx)}{1+\exp(w^Tx)}=P_i\tag{4} \\
  & p(y_i=0|x_i,w)=\dfrac{1}{1+\exp(w^Tx)}=1-P_i \tag{5}
\end{align}
 然后有：
\begin{align}
 & w^*=arg \  \underset{w}{max}P(D|w)\tag{16} \\
&\hspace {6mm}  =arg \  \underset{w}{max}\prod_{i=1}^NP_i^{y_i}(1-P_i)^{1-y_i} \tag{6} \\
& \hspace {6mm}  =arg \ \underset{w}{max}\sum_{i=1}^Ny_i\log{P_i}+(1-y_i)\log{(1-P_i)} \tag{7} \\
& \hspace {6mm}  =arg \  \underset{w}{max}\sum_{i=1}^Ny_i\log{\dfrac{P_i}{1-P_i}}+\log{(1-P_i)} \tag{8} \\
& \hspace {6mm}  =arg \  \underset{w}{max}=\sum_{i=1}^N[y_i\cdot w^Tx_i-\log{(1+\exp{(w^Tx_i)})}] \tag{9}
\end{align}
 据Andrew Ng关于最大似然函数与最小损失函数的关系: 
\begin{align}
 &  $J(\theta)=-\dfrac{1}{m}L(\theta) \tag{10}\\
\end{align}
 这里取:
\begin{align}
 &  J(w)=-L(w)=-\sum_{i=1}^N[y_i\cdot w^Tx_i-\log{(1+\exp{(w^Tx_i)})}] \tag{11}\\
\end{align}
 因此
\begin{align}
 & \dfrac{\partial J(w)}{\partial{w}}=-\sum_{i=1}^N[y_i\cdot x_i-\dfrac{\exp{(w^Tx_i)}}{1+\exp{(w^Tx_i})}\cdot{x_i}]\tag{12} \\
  & \hspace {14mm} =\sum_{i=1}^N(P_i-y_i)\cdot{x_i}  \tag{13}
\end{align}
 因此有参数的迭代如下：
 \begin{align}
& w_{j+1}=w_j-\alpha\cdot\sum_{i=1}^N(P_i-y_i)\cdot{x_i}  \tag{13}
\end{align}				
### 最大后验估计MAP与正则化

避免过拟合，降低server负担
 \begin{align}
 & w^*=arg \  \underset{w}{max}P(w|D) \tag{14} \\
  & \hspace {6mm}  = arg \  \underset{w}{max}P(w|D) \cdot P(D)\tag{15} \\
   & \hspace {6mm}  = arg \  \underset{w}{max}P(D|w) \cdot P(w)\tag{16} \\
    & \hspace {6mm}  = arg \  \underset{w}{max}[\log{P(D|w)} +\log{P(w)}]\tag{17} \\
\end{align}		



###在线训练
spark mllib, tmllib


### 参数调优

###效果对比
上面的工作目的都只有一个，把合适的内容推荐给用户，提高点击率。一般我们把请求服务器的用户分成一定比例的对照用户，对比算法的实际效果。典型的例如在算法、强规则、随机包、热销榜之间的点击流对比。
随机包就是指用户请求我们时，随机推荐攻略给用户；
强规则在各个业务中有所差异，例如在掌盟中将用户最近对局常失败的英雄对应的攻略推荐给用户；
热销榜则是根据最近12小时内的攻略点击率排行榜给用户推荐，热销榜具有很强的时效性，另外需要衡量点击率的指标需要略做优化。例如攻略A的曝光量是200，点击量是100，点击率是50%；同时攻略B的曝光量是2000，而点击量是900，点击率是45%，攻略A和B该怎么排序？这个也是需要注意的。
###贝叶斯平滑
预估互联网广告的点击率一个重要的技术手段是logistic regression模型，这个模型非常依赖特征的设计。每个广告的反馈ctr作为特征能极大地提升预估的准确性，所以每个广告的反馈ctr非常重要。
目前用得比较多的获取反馈ctr的方式是直接计算每个广告的历史ctr，这样的问题就是当该广告投放量比较少的时候（如新广告），历史ctr与实际ctr相差很大。如一个广告投放了100次，有2次点击，那么ctr就是2%，但是当这个广告投放量到了1000次的时候，点击只有10次，点击率是1%，这里就相差了一倍了。产生这种问题的的原因是投放量太少，数据有偏，所以如果每个广告在开始投放前就有了默认的一个展示数和点击数，即分子分母都加上一个比较大的常数，这样计算起ctr来就不会有那么大的偏差。这种方法叫做ctr平滑，通用的方法是在展示数和点击上面各自加一个常数，缓解低投放量带来的不准确性，使其接近其实际的CTR。


##推荐系统
实时推荐系统方面，我的同事在[《游戏广告推荐服务器原理介绍和实现总结》](http://km.oa.com/group/25372/articles/show/238608)中形象的介绍了实现的流程，在与推荐系统对接时，需要添加一些人工规则，例如最近的攻略，或者对模型的参数进行变化，来降低实时推荐系统的计算压力。能离线计算好的部分尽量先行计算，然后异步更新到Cache中，方便后续的计算、排序和推荐。

###EPR页面生成
![Alt text](./1469454293150.png)
EPR是互娱运营部数据中心推出的一套报表系统，旨在通过在EPR配置端创建报表和配置报表参数，可以将结构化的数据转换成成可视化报表并展现给用户。通过对报表页参数不同的配置，用户可以在浏览器中看到丰富的报表展现形式，如折线图、柱状图、表格等。具体使用方法可以参考[《使用“TDW+洛子系统+EPR”完成数据展示》](http://km.oa.com/group/18997/articles/show/220812?kmref=search&from_page=1&no=2&is_from_iso=1)

###监控告警
集群托管
不仅仅是在攻略推荐，还有各种游戏内的精准营销活动。
玩家的每一条日志数据对我们都很重要，在后续的拉新、拉活跃、拉付费、拉留存与防流失等各个环节中，玩家的每一个行为都很重要，玩家的每一次点击和付费行为数据都需要可靠保存和分析。

##待优化点
处理流程较长，
>目前用户属性是洛子调度写入tdw，然后出库至hdfs，配置mapreduce命令行进行离散化和dummy化并生成用户特征，然后批量写入tcaplus。由于用户量通常比较大，计算任务之间的依赖在洛子调度的时延可能会被放大，后面计划将用户属性部分改成spark来计算。
>针对攻略属性，细节特征。

##致谢
本文提及的攻略推荐工作是在数据挖掘组很多同事大量前期工作和丰富经验的基础上才得以完成。主要是在杜博、caron、sophie和woli等的帮助和指导下，不断发现和解决一个又一个细节问题。总结本文的目的在于梳理工作、找出潜在问题并推广至其他的业务。

##参考文献
[1、[《图灵系统文集》](http://km.oa.com/knowledge/2074)



