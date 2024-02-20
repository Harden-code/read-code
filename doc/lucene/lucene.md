索引

org.apache.lucene.document.Field

Field.Store.NO：不存储这个字段的值，但是还是会将这个值的文本进行切词后放入倒排索引中

Field.Store.YES：参数索引存储这个字段的值

- TextField:  Reader or  String indexed for full-text search 建立倒排索引
- StringField: String indexed verbatim as a single token 不分词
- IntPoint:  int indexed for exact/range queries.
- LongPoint:  long indexed for exact/range queries.
  - > LongPoint是Lucene中的一个字段类型，用于存储长整型（long）的字段值。它适用于需要对长整型字段进行范围查询的场景。使用LongPoint可以将长整型字段的值存储在倒排索引中，以便快速地进行范围查询。它可以用于存储和搜索长整型字段的值。以下是一些使用LongPoint的场景示例：- 范围查询：通过使用LongPoint和LongRangeQuery，可以对长整型字段进行范围查询，例如查找在某个长整型范围内的文档。- 过滤器：可以将LongPoint用作过滤器，用于筛选出符合特定长整型条件的文档。- 聚合操作：通过使用LongPoint，可以对长整型字段进行聚合操作，例如计算长整型字段的总和、平均值等。需要注意的是，使用LongPoint需要在建立索引时将长整型字段的值存储为LongPoint类型，并在搜索时使用相应的查询类进行操作。
- FloatPoint:  float indexed for exact/range queries.
- DoublePoint:  double indexed for exact/range queries.
- SortedDocValuesField:  byte[] indexed column-wise for sorting/faceting
- SortedSetDocValuesField:  SortedSet<byte[]> indexed column-wise for sorting/faceting
  - > SortedSetDocValuesField是Lucene中的一个字段类型，用于存储有序集合类型的字段值。它适用于需要对有序集合字段进行排序、范围查询或聚合操作的场景。使用SortedSetDocValuesField可以将有序集合字段的值存储在倒排索引中，以便快速地进行排序和范围查询。它可以存储多个值，并且这些值会按照字典顺序进行排序。以下是一些使用SortedSetDocValuesField的场景示例：- 对文档进行按有序集合字段排序：通过将有序集合字段存储为SortedSetDocValuesField，可以在搜索结果中按照有序集合字段的值进行排序。- 进行范围查询：通过使用SortedSetDocValuesRangeQuery，可以对有序集合字段进行范围查询，例如查找在某个值范围内的文档。- 进行聚合操作：通过使用SortedSetDocValuesField，可以对有序集合字段进行聚合操作，例如计算有序集合字段的并集、交集等。需要注意的是，使用SortedSetDocValuesField需要在建立索引时将有序集合字段的值存储为SortedSetDocValuesField类型，并在搜索时使用相应的查询类进行操作。
- NumericDocValuesField: long indexed column-wise for sorting/faceting
  - > NumericDocValuesField是Lucene中的一个字段类型，用于存储数值类型的字段值。它适用于需要对数值字段进行排序、范围查询或聚合操作的场景。使用NumericDocValuesField可以将数值字段的值存储在倒排索引中，以便快速地进行排序和范围查询。它可以存储整数类型（如int、long）和浮点数类型（如float、double）的字段值。以下是一些使用NumericDocValuesField的场景示例：- 对文档进行按数值字段排序：通过将数值字段存储为NumericDocValuesField，可以在搜索结果中按照数值字段的值进行排序。- 进行范围查询：通过使用NumericRangeQuery，可以对数值字段进行范围查询，例如查找在某个数值范围内的文档。- 进行聚合操作：通过使用NumericDocValuesField，可以对数值字段进行聚合操作，例如计算数值字段的总和、平均值等。需要注意的是，使用NumericDocValuesField需要在建立索引时将数值字段的值存储为NumericDocValuesField类型，并在搜索时使用相应的查询类进行操作。
- SortedNumericDocValuesField:  SortedSet<long> indexed column-wise for sorting/faceting
  - > SortedNumericDocValuesField是Lucene中的一个字段类型，用于存储有序数值类型的字段值。它适用于需要对有序数值字段进行排序、范围查询或聚合操作的场景。使用SortedNumericDocValuesField可以将有序数值字段的值存储在倒排索引中，以便快速地进行排序和范围查询。它可以存储多个数值，并且这些数值会按照升序进行排序。以下是一些使用SortedNumericDocValuesField的场景示例：- 对文档进行按有序数值字段排序：通过将有序数值字段存储为SortedNumericDocValuesField，可以在搜索结果中按照有序数值字段的值进行排序。- 进行范围查询：通过使用SortedNumericDocValuesRangeQuery，可以对有序数值字段进行范围查询，例如查找在某个数值范围内的文档。- 进行聚合操作：通过使用SortedNumericDocValuesField，可以对有序数值字段进行聚合操作，例如计算有序数值字段的总和、平均值等。需要注意的是，使用SortedNumericDocValuesField需要在建立索引时将有序数值字段的值存储为SortedNumericDocValuesField类型，并在搜索时使用相应的查询类进行操作。
- StoredField: Stored-only value for retrieving in summary results  它只存储字段的值，不参与检索

搜索

org.apache.lucene.search.Query

返回结果会按照匹配score进行倒排 **

- TermQuery 关键字查询 
  - > 关键词 就是指不可分割的整体,查找时会直接拿这个词去匹配倒排索引
- BooleanQuery 复合查询
  - BooleanClause.Occur
    - MUST
    - MUST_NOT 需要遍历所有的文档，无法直接使用倒排索引，会严重影响查询效率
    - SHOULD  会影响排序结果而不会影响命中
    - FILTER FILTER 会影响查询结果但是不影响排序，它只起到过滤的作用，就好比数据库查询里的 Where 条件
- WildcardQuery：模糊查询
- PhraseQuery：短语匹配
- PrefixQuery 前缀查询，不会对搜索的结果进行排序，所有被搜索出来的文档统一打分 1.0，在实现上可以让查询效率快很多，直接省去了收集所有文档进行排序的过程
  - > 默认关闭,前缀查询能匹配到的关键词可能会很多,merge 所有的文档列表并排序将会是一个非常耗费性能的过程。
- MultiPhraseQuery
- FuzzyQuery 编辑距离
- RegexpQuery：正则查询
- TermRangeQuery：字符串范围查询 TermRangeQuery.newStringRange("level", "level1", "level9", true, true);bool表示边界
- PointRangeQuery
- ConstantScoreQuery
- DisjunctionMaxQuery
- MatchAllDocsQuery：全表遍历查询，不走索引
- NumericRangeQuery：数字范围查询默认索引不存储,IntPoint.newRangeQuery("id", 400000, 450000); 

tutorials

https://www.baeldung.com/lucene

Data Structure

https://cwiki.apache.org/confluence/pages/viewrecentblogposts.action?key=LUCENE

https://www.joinquant.com/view/community/detail/c2c41c79657cebf8cd871b44ce4f5d97

How to work

https://stackoverflow.com/questions/2705670/how-does-lucene-work

http://infolab.stanford.edu/~backrub/google.html

**t**erm **f**requency–**i**nverse **d**ocument **f**requency

https://zh.wikipedia.org/wiki/Tf-idf

Luke Utils

https://github.com/DmitryKey/luke

Analyzer

https://code.google.com/archive/p/ik-analyzer/downloads

#### Scoring

These models can be plugged in via the `Similarity API`, and offer extension hooks and parameters for tuning. In general, Lucene first finds the documents that need to be scored based on boolean logic in the Query specification, and then ranks this subset of matching documents via the retrieval model. 

- [Vector Space Model (VSM)](http://en.wikipedia.org/wiki/Vector_Space_Model)
- [Probabilistic Models](http://en.wikipedia.org/wiki/Probabilistic_relevance_model) such as [Okapi BM25](http://en.wikipedia.org/wiki/Probabilistic_relevance_model_(BM25)) and [DFR](http://en.wikipedia.org/wiki/Divergence-from-randomness_model)
- [Language models](http://en.wikipedia.org/wiki/Language_model)

> ##### Information Retrieval
>
> https://en.wikipedia.org/wiki/Vector_space_model
>
> https://www-nlp.stanford.edu/IR-book/
>
> https://nlp.stanford.edu/IR-book/html/htmledition/scoring-term-weighting-and-the-vector-space-model-1.html

###### Vector space model

In this section we consider a particular vector space model based on the [bag-of-words](https://en.wikipedia.org/wiki/Bag-of-words_model) representation. Documents and queries are represented as vectors.

Each [dimension](https://en.wikipedia.org/wiki/Dimension_(vector_space)) corresponds to a separate term. If a term occurs in the document, its value in the vector is non-zero. Several different ways of computing these values, also known as (term) weights, have been developed. One of the best known schemes is [tf-idf](https://en.wikipedia.org/wiki/Tf-idf) weighting (see the example below).

The definition of *term* depends on the application. Typically terms are single words, [keywords](https://en.wikipedia.org/wiki/Keyword_(linguistics)), or longer phrases. If words are chosen to be the terms, the dimensionality of the vector is the number of words in the vocabulary (the number of distinct words occurring in the [corpus](https://en.wikipedia.org/wiki/Text_corpus)).

Vector operations can be used to compare documents with queries.

> corresponds 类似
>
> separate 不同 occurs 出现

#### Scoring — Basics

Scoring is very much dependent on the way documents are indexed, so it is important to understand indexing. Be sure to use the useful `IndexSearcher.explain(Query, doc)` to understand how the score for a certain matching document was computed.

Generally, the Query determines which documents match (a binary decision), while the Similarity determines how to assign scores to the matching documents.

```
1.1889626 = weight(keyword:无名氏 in 936) [BM25Similarity], result of:
  1.1889626 = score(freq=1.0), product of:
    2.6120467 = idf, computed as log(1 + (N - n + 0.5) / (n + 0.5)) from:
      163 = n, number of documents containing term
      2227 = N, total number of documents with field
    0.45518428 = tf, computed as freq / (freq + k1 * (1 - b + b * dl / avgdl)) from:
      1.0 = freq, occurrences of term within document
      1.2 = k1, term saturation parameter
      0.75 = b, length normalization parameter
      3.0 = dl, length of field
      3.0103278 = avgdl, average length of field
```

> Determines 确定

#### *Fields and Documents*

In Lucene, the objects we are scoring are `Document`s. A Document is a collection of `Field`s. Each Field has `semantics` about how it is created and stored (`tokenized`, `stored`, etc). It is important to note that Lucene scoring works on Fields and then combines the results to return Documents. This is important because two Documents with the exact same content, but one having the content in two Fields and the other in one Field may return different scores for the same query due to length normalization.

> Semantics 语义 due 由于

##### *Changing the scoring formula*

###### Changing `Similarity` 

....

###### *Integrating field values into the score*

 Such features are best integrated into the score by indexing a `FeatureField` with the document at index-time, and then combining the similarity score and the feature score using a linear combination. For instance the below query matches the same documents as `originalQuery` and computes scores as `similarityScore + 0.7 * featureScore`

```
 Query originalQuery = new BooleanQuery.Builder()
     .add(new TermQuery(new Term("body", "apache")), Occur.SHOULD)
     .add(new TermQuery(new Term("body", "lucene")), Occur.SHOULD)
     .build();
 Query featureQuery = FeatureField.newSaturationQuery("features", "pagerank");
 Query query = new BooleanQuery.Builder()
     .add(originalQuery, Occur.MUST)
     .add(new BoostQuery(featureQuery, 0.7f), Occur.SHOULD)
     .build();
```

> formula 公式
>
> integrated  综合

#### Custom Queries — Expert Level