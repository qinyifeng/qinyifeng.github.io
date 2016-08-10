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
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
