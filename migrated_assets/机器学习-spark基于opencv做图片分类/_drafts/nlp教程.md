---
title: nlp教程
date: 2018-03-28 09:39:17
tags:
---



## 英文文本预处理的特点
英文文本的预处理方法和中文的有部分区别。首先，英文文本挖掘预处理一般可以不做分词（特殊需求除外），
而中文预处理分词是必不可少的一步。第二点，大部分英文文本都是uft-8的编码，
这样在大多数时候处理的时候不用考虑编码转换的问题，
而中文文本处理必须要处理unicode的编码问题。这两部分我们在中文文本挖掘预处理里已经讲了。

而英文文本的预处理也有自己特殊的地方，第三点就是拼写问题，很多时候，我们的预处理要包括拼写检查，
比如“Helo World”这样的错误，我们不能在分析的时候讲错纠错。
所以需要在预处理前加以纠正。
第四点就是词干提取(stemming)和词形还原(lemmatization)。这个东西主要是英文有单数，复数和各种时态，
导致一个词会有不同的形式。比如“countries”和"country"，"wolf"和"wolves"，我们期望是有一个词。

1. 处理拼写错误
2. 词干提取


### 步骤
  1. 准备好语料库
  2. 拼写检查
  3. 词干提取(stemming)和词形还原(lemmatization)
  #### 一次nlp作业
  1. 语料库的丰富程度决定了最后使用词袋模型的准确率效果
  2. tfidf 之后可以考虑 做PCA主成分分析法
  3.  在大规模的文本处理中，由于特征的维度对应分词词汇表的大小，所以维度可能非常恐怖，
  此时需要进行降维，不能直接用我们上一节的向量化方法。而最常用的文本降维方法是Hash Trick。
  在Hash Trick里，我们会定义一个特征Hash后对应的哈希表的大小，这个哈希表的维度会远远小于我们的词汇表的特征维度，因此可以看成是降维。
  具体的方法是，对应任意一个特征名，我们会用Hash函数找到对应哈希表的位置，然后将该特征名对应的词频统计值累加到该哈希表位置。

  采用scikit learn 中的 hashingvectorizer
  Convert a collection of text documents to a matrix of token occurrences
  This strategy has several advantages:
it is very low memory scalable to large datasets as there is no need to store a vocabulary dictionary in memory
it is fast to pickle and un-pickle as it holds no state besides the constructor parameters
it can be used in a streaming (partial fit) or parallel pipeline as there is no state computed during fit.

The hash function employed is the signed 32-bit version of Murmurhash3.

### 中文自然语言处理
#### 中文分词

