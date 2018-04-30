---
layout: default
title: StanfordCoreNLP
date: 2018-04-06
categories: NLP
tags: [NLP]
description: StanfordCoreNLP是一个自然语言处理的开源项目，支持多种语言。
---

###  下载与安装
StanfoCoreNLP需要[java8运行环境](https://www.java.com)，可以在[官方网站](https://stanfordnlp.github.io/CoreNLP/index.html#download)下载最新版本的软件包。<br>
也可以在下载页面，下载其他语言包来获得对应语言的支持。

###  运行
在解压后的文件夹内打开命令行，输入

`java -mx4g -cp "*" edu.stanford.nlp.pipeline.St
anfordCoreNLPServer -port 9000 -timeout 15000`
`mx4g`表示最大内存4G，官方建议为pipeline分配至少2G内存；`cp "*"`表示载入当前目录的所有jar文件；`-port 9000`开启端口为9000； `timeout 15000`超时时间15000毫秒；

可以添加`-serverProperties StanfordCoreNLP-chinese.properties`来使用中文语言包。

该命令在本机的9000端口开启`StanfordCoreNLPServer`，我们可以去访问这个网页，也可以POST数据来得到返回结果。

`wget --post-data 'The quick brown fox jumped over the lazy dog.' 'localhost:9000/?properties={"annotators":"tokenize,ssplit,pos","outputFormat":"json"}' -O -`

###  Annotators
StanfordCoreNLP支持多种Annotator来对文本进行处理

|Annotator|Description|
|-|-|
|tokenize|把给定文本分成词语
|cleanxml|移除XML文件的标签
|ssplit|把一串词划分成句子
|pos|标明每个分词的词性
|lemma|确定词元
|ner|命名实体识别，比如人名，地名，数字，日期等
|regexner|通过java正则表达式识别更多实体
|sentiment|情感
|truecase|正确转化大小写
|parse|语法分析
|depparse|语法依赖性分析
|dcoref|指代，人称代词分析
|relation|两个实体间的关系
|natlog|根据自然逻辑语义，标记量化范围和实体极性
|quote|挑出句子中的引用成分

###  PyCoreNLP
调用Python的`pycorenlp`模块来使用Core NLP的API

安装

`pip install pycorenlp`


```
>>>from pycorenlp import StanfordCoreNLP
>>>nlp = StanfordCoreNLP('http://localhost:9000')
>>>text = (
        'Pusheen and Smitha walked along the beach. Pusheen wanted to surf,'
        'but fell off the surfboard.')
>>>output = nlp.annotate(text, properties={
        'annotators': 'tokenize,ssplit,pos,depparse,parse',
        'outputFormat': 'json'
    })
>>>print(output['sentences'][0]['parse'])
```


