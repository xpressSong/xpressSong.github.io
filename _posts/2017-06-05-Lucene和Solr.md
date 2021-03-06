---  
layout: post  
title: Lucene和Solr  
tags: Lucene Solr  
categories: JavaEE  
published: true  
---  

## 信息检索与全文检索

信息检索是从信息集合中找出与用户需求相关的信息，包括文本、图像、视频、音频等多媒体信息

* 全文检索：把用户的查询需求和全文中的每一个分词进行比较，不考虑查询请求与文本语义上的匹配，最具通用性和实用性
* 数据检索：查询要求和信息系统中的数据遵循一定的格式，具有一定的结构，允许特定的字段检索，其性能与使用有很大的局限性，支持语义匹配的能力也很差
* 知识检索：强调基于知识的、语义上的匹配

### 全文索引

#### 建立索引

* 信息采集：把信息源的信息拷贝到本地，构成待检索的信息集合
* 信息加工；为爱刺激到本地的信息编排索引，为查询做好准备

![index](/static/img/Lucene/index.png "index")

#### 分词器

* 文本在建立索引和搜索时都会先分词
* 为了保证能正确搜索到结果，在建立索引与进行搜索时使用的分词器应该是同一个

![keyAnalyzer](/static/img/Lucene/keyAnalyzer.png "keyAnalyzer")

#### 分词步骤

##### 英文分词

![en_analyzer](/static/img/Lucene/en_analyzer.PNG "en_analyzer")

1. 首先切分词
2. 排除停用词：标点符号和“的、了、着、a、an、of”等词语，排除停用词可以加快建立索引的速度，还可以减少索引文件的大小
3. 形态还原：期初单词词尾的形态变化，将其还原为词的原形（worked-work）
4. 转换为小写

##### 中文分词器

* 单词分词：一个字分成一个词（StandardAnalyzer）
* 二分法分词：按每两个字进行切分（CJKAnalyzer）
* 词典分词：按照某种算法构造词，去匹配已经建好的词库集合，匹配到就分为词语（MMAnalyzer极易中文分词、庖丁分词、IK Analyzer等）  


#### 索引结构

* 索引文件结构是倒排索引，索引对象是文档中的单词，用来存储这些单词在文档中的位置
* 索引库时一组文本的集合，索引库位置Directory
* 文档Document是Filed的集合，Field的值是文本

![Directory](/static/img/Lucene/Directory.png "Directory")

* 进行检索时，先从检索词汇表（索引表）开始，然后找到相对应的文档
* 倒排索引的维护一般采用先删除后创建的方式替代更新操作，更新操作代价较高

## Lucene

### 创建索引

![store](/static/img/Lucene/store.png "store")

```java
public static final Version LUCENE_43 = Version.LUCENE_43;

public void indexDoc(String directoryPath) throws IOException {
    Directory directory = new SimpleFSDirectory(new File(directoryPath));
    Analyzer analyzer = new SimpleAnalyzer(LUCENE_43);
    IndexWriterConfig indexWriterConfig = new IndexWriterConfig(LUCENE_43, analyzer);
    // prepare directory
    IndexWriter indexWriter = new IndexWriter(directory, indexWriterConfig);
    // prepare document
    ArrayList<IndexableField> indexableFields = new ArrayList<IndexableField>();
    // prepare fields
    IndexableField name = new TextField("name", "c++", Field.Store.YES);
    IndexableField author = new TextField("author", "bob smith", Field.Store.YES);
    IndexableField desc = new TextField("desc", "this is a c book", Field.Store.YES);
    IndexableField size = new StringField("size", "13kb", Field.Store.YES);
    // custom index and store
    FieldType fieldType = new FieldType();
    fieldType.setIndexed(false);// not index
    fieldType.setStored(true);// stored
    IndexableField content = new Field("content", "this is a c++ book content!!!!", fieldType);
    indexableFields.add(name);
    indexableFields.add(author);
    indexableFields.add(desc);
    indexableFields.add(size);
    indexableFields.add(content);
    // store
    indexWriter.addDocument(indexableFields);
    indexWriter.close();
}
```

#### Field.Store 存储域选项

* Field.Store.YES
    - 完全把这个域中的内容完全存储到文件中，可以还原文件内容
* Field.Store.NO
    - 不存储这个域中的内容到文件中，但是不影响索引，无法还原文件内容

#### Field.Index 索引选项

* Index.ANALYZED 进行分词和索引
* Index.NOT_ANALYZED 进行索引，但是不进行分词
* Index.ANALYZED_NOT_NORMS 进行分词但是不存储norms信息，这个norms信息包含了索引时间和权值等
* Index.NOT_ANALYZED_NOT_NORMS 既部分词也不存储norms信息
* Index.NO 不进行索引

**4.0以后Field.Index废弃**

* StringField 不分词并索引的字段，搜索时整字段匹配
* TextField 分词并索引
* StoredField 仅保存，不索引

另外，权重信息可以在Field中指定

```java
FieldType type = new FieldType();
type.setIndexed(true);
type.setStored(true);
type.setIndexOptions(FieldInfo.IndexOptions.DOCS_AND_FREQS);
indexableFields.add(new Field("content", content, type));
```

* FieldInfo.IndexOptions.DOCS_ONLY 索引文档编号
* FieldInfo.IndexOptions.DOCS_AND_FREQS 索引文档编号、关键词频率
* FieldInfo.IndexOptions.DOCS_AND_FREQS_AND_POSITIONS 索引文档编号、关键词频率、位置
* FieldInfo.IndexOptions.DOCS_AND_FREQS_AND_POSITIONS_AND_OFFSETS 索引文档编号、关键词频率、位置、每个关键字偏移量

#### 索引库

* 文件系统索引库；FSDirectory
* 内存索引库：RAMDirectory

```java
public void directoryTest(String directoryPath) throws IOException {
    Directory fsDirectory = new SimpleFSDirectory(new File(directoryPath));
    // 启动时加载文件系统索引库到系统内存索引库
    Directory ramDirectory = new RAMDirectory(fsDirectory, IOContext.DEFAULT);
    IndexWriterConfig indexWriterConfig = new IndexWriterConfig(LUCENE_43, new SimpleAnalyzer(LUCENE_43));
    IndexWriter indexWriter = new IndexWriter(fsDirectory, indexWriterConfig);
    // 退出时保存
    indexWriter.addIndexes(new Directory[]{ramDirectory});
    indexWriter.commit();
    indexWriter.close();
}
```

##### 索引库文件

* fnm文件 存储哪些域信息
* fdt、fdx文件 Store.YES的域内容
* frq文件 出现次数
* nrm文件 评分信息
* prx文件 偏移量
* tii、tis 文件 索引信息

### 查询索引

#### DirectoryReader

##### openIfChanged方式

```java
private static DirectoryReader DIRECTORY_READER;
private static final String INDEX_PATH = "";
public static synchronized DirectoryReader getReader() {
    try {
        if (DIRECTORY_READER == null) {
            DIRECTORY_READER = DirectoryReader.open(new SimpleFSDirectory(new File(INDEX_PATH)));
        } else {
            // 如果有变化重新打开一个 使变更生效
            DirectoryReader indexReader = DirectoryReader.openIfChanged(DIRECTORY_READER);
            if (indexReader != null) {
            	DIRECTORY_READER.close();
                DIRECTORY_READER = indexReader;
            }
        }
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
    return DIRECTORY_READER;
}
```

```java
public void searchDoc(String[] keywords, String directoryPath) throws IOException {
    Directory directory = new SimpleFSDirectory(new File(directoryPath));
    IndexReader indexReader = DirectoryReader.open(directory);
    System.out.println(indexReader.numDocs());// 文档总数
    System.out.println(indexReader.numDeletedDocs());// 删除文档总数
    System.out.println(indexReader.maxDoc());// 存储过的做大存储量
    // init directory
    IndexSearcher indexSearcher = new IndexSearcher(indexReader);
    // query
    MultiPhraseQuery query = new MultiPhraseQuery();
    query.add(new Term(keywords[0], keywords[1]));
    // filter
    Filter filter = null;
    TopDocs topDocs = indexSearcher.search(query, filter, 1000);// limit 1000
    // count
    System.out.println(topDocs.totalHits);
    for (ScoreDoc scoreDoc : topDocs.scoreDocs) {
        // get doc
        Document doc = indexSearcher.doc(scoreDoc.doc);
        System.out.println(doc.get("name"));
        System.out.println(doc.get("author"));
        System.out.println(doc.get("desc"));
        System.out.println(doc.get("size"));
        System.out.println(doc.get("content"));
        System.out.println("-------------");
    }
}
```

##### SearchManager和近实时搜索

```java
private static SearcherManager searcherManager;
private static TrackingIndexWriter trackingIndexWriter;
private static ControlledRealTimeReopenThread controlledRealTimeReopenThread;

static {
    try {
        INDEX_PATH = SearchService.class.getClassLoader().getResource("indexes").getPath();
        SimpleFSDirectory dir = new SimpleFSDirectory(new File(INDEX_PATH));
        IndexWriter indexWriter = new IndexWriter(dir, new IndexWriterConfig(Version.LUCENE_43, new IKAnalyzer(true)));
        indexWriter.commit();// 建立索引库基本信息防止searcherManager初始化报错

        // 初始化SearcherFactory
        SearcherFactory searcherFactory = new SearcherFactory();
        searcherManager = new SearcherManager(dir, searcherFactory);

        //ControlledRealTimeReopenThread 构造是必须有的，主要将writer封装，每个方法都没有commit 操作。
        trackingIndexWriter = new TrackingIndexWriter(indexWriter);
        //创建线程，用于管理writer和searcher的近实时同步
        double targetMaxStaleSec = 5.0;
        double targetMinStaleSec = 0.025;
        controlledRealTimeReopenThread = new ControlledRealTimeReopenThread<IndexSearcher>(trackingIndexWriter, searcherManager, targetMaxStaleSec, targetMinStaleSec);
        controlledRealTimeReopenThread.setDaemon(true);//设为后台进程
        controlledRealTimeReopenThread.setName("定时清理缓存");
        controlledRealTimeReopenThread.start();//启动线程

    } catch (IOException e) {
        e.printStackTrace();
    }
}

public void search(Query query) throws IOException {
    //searcherManager.maybeRefresh(); // 刷新由线程管理
    IndexSearcher indexSearcher = searcherManager.acquire();
    //...
    searcherManager.release(indexSearcher);
}

public void add(Query query) throws IOException {
    trackingIndexWriter.addDocument(document);
}

@Override
protected void finalize() throws Throwable {
    super.finalize();
    controlledRealTimeReopenThread.close();
    commitChanges();
    searcherManager.close();
}

// 用于检查可能的更新操作,定时刷新保证效率
@Scheduled
public void commitChanges() throws IOException {
    trackingIndexWriter.getIndexWriter().commit();
}
```

#### Query

##### TermQuery 关键词查询

```java
public void termQueryTest() throws Exception {
    // Term term = new Term("content", "JAVA"); // 英文关键词查询只有小写，没有大写，因为存储时会转换为小写
    Term term = new Term("content", "java");
    Query query = new TermQuery(term);
    indexAndSearchService.searchDoc(query, directoryPath);
}
```

##### TermRangeQuery和NumericRangeQuery 范围查询

```java
public void rangeQueryTest() throws Exception {
    boolean includeLower = true;
    boolean includeUpper = true;
    // supplied range according to Byte.compareTo(Byte).数字2要补0才会算出数字效果
    // BytesRef lowerTerm = new BytesRef("02");
    // BytesRef upperTerm = new BytesRef("20");
    // Query query = new TermRangeQuery("size", lowerTerm, upperTerm, includeLower, includeUpper);
    // 使用数字限定范围
    NumericRangeQuery query =  NumericRangeQuery.newIntRange("order", 0, 77, includeLower, includeUpper);
    indexAndSearchService.searchDoc(query, directoryPath);
}
```

##### WildcardQuery 通配符查询

* `*`：匹配任意个任意字符
* `?`：匹配一个任意字符

```java
Query query = new WildcardQuery(new Term("name","c*"));
Query query = new WildcardQuery(new Term("name","ja??"));
searchService.searchByQuery(query, indexPath);
```

##### PhraseQuery 短语查询

```java
PhraseQuery query = new PhraseQuery();
// 可以指定短语位置
// query.add(new Term("content", "c", 5));
query.add(new Term("content", "c"));
query.add(new Term("content", "指针"));
query.setSlop(20);// 最大间词语个数
searchService.searchByQuery(query, indexPath);
```

##### BooleanQuery 组合查询

BooleanClause.Occur

* MUST和MUST：取交集
* MUST和MUST_NOT：取差集
* SHOULD和SHOULD：取并集

* MUST和SHOULD：结果为MUST结果
* MUST_NOT和MUST_NOT：无意义，无结果
    - MUST_NOT：单独使用无意义，无结果
* MUST_NOT和SHOULD：取差集
    - SHOULD：单独使用同MUST

```java
PhraseQuery query1 = new PhraseQuery();
query1.add(new Term("content", "对象"));
query1.add(new Term("content", "指针"));
query1.setSlop(20);// 最大间词语个数
// Query query = new WildcardQuery(new Term("content","对象"));
// Query query = new WildcardQuery(new Term("name","ja??"));
Query query2 = new TermQuery(new Term("name","kotlin"));
// Query query = new TermQuery(new Term("name", "j"));

BooleanQuery query = new BooleanQuery();
query.add(query1, BooleanClause.Occur.MUST);
query.add(query2, BooleanClause.Occur.MUST);
searchService.searchByQuery(query, indexPath);
```

##### FuzzyQuery 模糊查询

FuzzyQuery的相似度计算使用的Damerau-Levenshtein，FuzzyQuery的transpositions属性设置为false表示启用Damerau-Levenshtein来计算相似度

```java
FuzzyQuery query = new FuzzyQuery(new Term("content","面前"));
searchService.searchByQuery(query, indexPath);
```

##### PrefixQuery 前缀查询

```java
PrefixQuery query = new PrefixQuery(new Term("name", "c"));
```

##### QueryParser

详情参看查询语法

```java
QueryParser queryParser = new QueryParser(Version.LUCENE_43, "name", new MMSegAnalyzer());
Query query = queryParser.parse("java c");
searchService.searchByQuery(query, indexPath);

// 更改默认分隔符 默认是or
queryParser.setDefaultOperator(QueryParser.Operator.AND);
// 设置首位通配符开关，因为效率很低默认关闭
queryParser.setAllowLeadingWildcard(true);
```

###### 自定义QueryParser

1. 对于某些(FuzzyQuery,WildcardQuery)查询会将查询效率降低，所以考虑将这些查询取消
2. 在具体的查询时，很有可能获取的是一个未提供功能（数字、日期）的范围查询，所以考虑扩展

``java
public class CustomQueryParser extends QueryParser {
    public CustomQueryParser(Version matchVersion, String f, Analyzer a) {
        super(matchVersion, f, a);
    }
    @Override
    protected Query getWildcardQuery(String field, String termStr) throws ParseException {
        // throw new ParseException("通配符查询已经禁用");
        return getFieldQuery(field, termStr, true);
    }
    @Override
    protected Query getRangeQuery(String field, String part1, String part2, boolean startInclusive, boolean endInclusive) throws ParseException {
        if ("order".equals(field)) {
            return NumericRangeQuery.newIntRange(field, Integer.valueOf(part1), Integer.parseInt(part2), startInclusive, endInclusive);
        }
        return super.getRangeQuery(field, part1, part2, startInclusive, endInclusive);
    }
}
```

#### DateTools

```java
DateTools.dateToString(new Date() , DateTools.Resolution.DAY);// 20170607
DateTools.dateToString(new Date() , DateTools.Resolution.HOUR);// 2017060707
DateTools.dateToString(new Date() , DateTools.Resolution.MILLISECOND);// 20170607073809118
DateTools.dateToString(new Date() , DateTools.Resolution.MINUTE);// 201706070738
```

#### 查询语法

```shell
# TermQuery
name:j
# TermRangeQuery
size:[02 TO 20]
# NumericRangeQuery
order:[1 TO 20]
# WildcardQuery
name:ja??
# PhraseQuery
content:"? 对象 ? 指针"
content:"对象 指针"
content:"对象 指针"~20
# BooleanQuery
+content:"对象 指针"~20 +name:kotlin
# 相关度
name:我^3.0 content:我^2.0
```

查询语法使用QueryParser与MultiFieldQueryParser进行解析

* `+`和`-`：表示后面条件是MUST还是MUST_NOT
* AND：MUST
* OR：SHOULD
* NOT:MUST_NOT
* 可以使用括号就行逻辑组合

#### 排序

* 相关度（默认），在搜索是指定Field的boost，默认1.0f
* 自定义排序，使用Sort，默认为生序

```java
// name:我^3.0 content:我^2.0
Map<String, Float> stringFloatMap = new HashMap<String, Float>();
stringFloatMap.put("name", 3.0f);// 指定boost，默认1.0f
stringFloatMap.put("content", 2.0f);
Analyzer analyzer = new MMSegAnalyzer();
MultiFieldQueryParser queryParser = new MultiFieldQueryParser(Version.LUCENE_43, new String[]{"name", "content"}, analyzer, stringFloatMap);
Query query = queryParser.parse("我是");
searchService.searchByQuery(query, indexPath);
```

```java
// Sort.RELEVANCE 根据分支排序
// Sort.INDEXORDER 根据索引号排序

// 根据不同的域进行排序
Sort sort = new Sort();
boolean reverse = false;// order字段降序排列
sort.setSort(new SortField("order", SortField.Type.INT, reverse));
TopDocs topDocs = indexSearcher.search(query, 200, sort);
```

#### 过滤器

对搜索结果进行过滤，效率低

```java
// 数字范围
Filter filter = NumericRangeFilter.newIntRange("order", 35, 45, true, true);
// 字符串范围
filter = new TermRangeFilter("name", new BytesRef("aaa"), new BytesRef("bbb"), true, true);
// 通配符
filter = new QueryWrapperFilter(new WildcardQuery(new Term("name", "j*")));
TopDocs topDocs = indexSearcher.search(query, filter, 200, sort);
```

##### 自定义Filter

```java
public class MyCustomFilter extends Filter {
    // private String[] blackList = new String[]{"19", "18", "17", "29", "117","114", "16", "15", "14"};
    private int[] blackList = new int[]{19, 18, 17, 29, 117, 16, 15, 14};
    @Override
    public DocIdSet getDocIdSet(AtomicReaderContext context, Bits acceptDocs) throws IOException {
        AtomicReader reader = context.reader();
        int maxDocs = reader.maxDoc();
        OpenBitSet openBitSet = new OpenBitSet(maxDocs);
        openBitSet.set(0, maxDocs);// 全部允许
        // for (String item : blackList) {
        for (int item : blackList) {
            // 数字类型处理（存储时numericType为INT）
            BytesRefBuilder bytesRefBuilder = new BytesRefBuilder();
            NumericUtils.intToPrefixCoded(item, 0, bytesRefBuilder);
            // 获取符合要求的字段并禁用
            DocsAndPositionsEnum prders = reader.termPositionsEnum(new Term("order", bytesRefBuilder.toBytesRef()));
            // DocsAndPositionsEnum prders = reader.termPositionsEnum(new Term("order", item));
            if (prders == null) {
                continue;
            }
            int i;
            while ((i = prders.nextDoc()) != NO_MORE_DOCS) {
                openBitSet.clear(i);// 删除黑名单
            }
        }
        return openBitSet;
    }
}
```

#### 评分

##### 内置评分

![score](/static/img/Lucene/score.jpg "score")

> d:doc,t:term,q:query,f:field

* 协调因子coord(q,d)：查询字段的命中个数
    - 查询名称和作者，名称命中，为1/2
* 查询规范因子queryNorm(q)：与q.getBoost()和t.getBoost()有关，比较两次查询，单次查询对排序无影响
* 文档词频因子tf(t in d)：词在文档中出现的次数开方
    - 查询名称，出现两次，为1.414
* 文档出现频率因子idf(t)：log(总文档数/(命中文档数+1))+1
* 查询权重t.getBoost()：termquery的boost值，默认1.0
* 标准化因子norm(t,d)：(1/分词个数的开方)*各个field的boost


> tf(t in d) 表示某个term的出现频率，定义了term t出现在当前document d的次数。 对于query中的term，出现的越多，得分就越高。  
> idf(t) 表示反向文档频率。这个参数表示docFreq(term t一共在多少个文档中出现)的反向影响值。它意味着在越少文档中出现的terms贡献的分数越高(物以稀为贵)。  
> coord(q,d) 是一个基于在该文档中出现了多少个query中的terms的得分因素。越多的查询项在一个文档中，说明些文档的匹配程度越高。默认是出现查询项的百分比。  
> queryNorm(q) 是一个标准化参数，使不同查询之间可以比较。此因子不影响文档的排序，因为所有有文档都会使用此因子。  
> t.getBoost() 是一个term 在query 中的搜索时间中的加权， 它在query中指定, 或者被应用程序直接调用setBoost()设置。  
> norm（t,d）是在索引时进行计算并存储的，在查询时是无法再改变的，除非再重建索引。norm值是被压缩存储的，在查询时取出该值进行文档相关度计算。

###### 可改变部分

* Api
    - 索引时刻：field boost
    - 查询时刻：query boost
* 重写源码
    - Similarity

**查看评分详情**

```java
indexSearcher.explain(query, scoreDoc.doc)

// 1.0 = (MATCH) ConstantScore(name:*c*), product of:
//   1.0 = boost
//   1.0 = queryNorm
```

##### 自定义评分

###### 拆分文档分数

1. 创建一个类继承CustomScoreQuery，覆盖getCustomScoreProvider()方法
2. 创建一个类继承CustomScoreProvider，覆盖customScore()方法

```java
public class MyCustomScoreQuery extends CustomScoreQuery {
    public MyCustomScoreQuery(Query subQuery) {
        super(subQuery);
    }
    @Override
    protected CustomScoreProvider getCustomScoreProvider(AtomicReaderContext context) throws IOException {
        return new MyCustomScoreProvider(context);
    }
    private class MyCustomScoreProvider extends CustomScoreProvider {
        private BinaryDocValues names;

        MyCustomScoreProvider(AtomicReaderContext context) {
            super(context);
            try {
                names = FieldCache.DEFAULT.getTerms(context.reader(), "name", true);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        @Override
        public float customScore(int doc, float subQueryScore, float valSrcScore) throws IOException {
            System.out.println("subQueryScore:" + subQueryScore + "valSrcScore:" + valSrcScore);
            float score = valSrcScore * subQueryScore;// 自定义评分计算规则
            BytesRef bytesRef = names.get(doc);
            String string = bytesRef.utf8ToString();
            System.out.println("string" + doc + ":" + string);
            if (string.contains("r")) {
                return score * 1.5f;// 根据自己的规则修改评分
            }
            return score;
        }
    }
}
```

```java
Map<String, Float> stringFloatMap = new HashMap<String, Float>();
stringFloatMap.put("name", 3.0f);
stringFloatMap.put("content", 2f);
Analyzer analyzer = new MMSegAnalyzer();
MultiFieldQueryParser queryParser = new MultiFieldQueryParser(Version.LUCENE_43, new String[]{"name", "content"}, analyzer, stringFloatMap);
queryParser.setAllowLeadingWildcard(true);
Query query = queryParser.parse("(name:*a* content:*c*) (name:*c* content:*y*) (name:*a* content:*k*) (name:*k* content:*k*)");
// 评分query
ConstantScoreQuery constantScoreQuery = new ConstantScoreQuery(query);
// 自定义评分query
MyCustomScoreQuery myCustomScoreQuery = new MyCustomScoreQuery(query, constantScoreQuery);
searchService.searchByQuery(myCustomScoreQuery, indexPath, 1, 22);
```

###### 自定义字段评分

#### 分页

> lucene3.5之前分页提供的方式为再查询方式（每次查询全部记录，然后取其中部分记录，这种方式用的最多），lucene官方的解释：由于我们的速度足够快。处理海量数据时，内存容易内存溢出。  
> lucene3.5以后提供一个searchAfter，这个是在特大数据量采用（亿级数据量），速度相对慢一点，像google搜索图片的时候，点击更多，然后再出来一批。这种方式就是把数据保存在缓存里面。然后再去取。

```java
// 再查询方式
int skip = (pageNum - 1) * pageSize;
int max = skip + pageSize;
TopDocs topDocs = indexSearcher.search(query, max, sort);
int totalHits = topDocs.totalHits;
System.out.println(totalHits);
max = (max > totalHits ? totalHits : max);// 计算总数
for (int i = skip; i < max; i++) {
    // 通过查询结果取出文档
    Document doc = indexSearcher.doc(topDocs.scoreDocs[i].doc);
    System.out.println(doc.get("name"));
    System.out.println(doc.get("author"));
    System.out.println("---------------");
}
```

```java
// searchAfter方式
int skip = (pageNum - 1) * pageSize;
int max = skip + pageSize;// 计算最大查询总数

// 首先获取after
TopDocs topDocs = indexSearcher.search(query, max, sort);

ScoreDoc after = null;
if (skip != 0) {
    after = topDocs.scoreDocs[skip - 1];
}
// 通过searchAfter完成分页
TopDocs result = indexSearcher.searchAfter(after, query, max, sort);
for (ScoreDoc scoreDoc : result.scoreDocs) {
    Document doc = indexSearcher.doc(scoreDoc.doc);
    System.out.println("BOOK:" + scoreDoc.doc);
    System.out.println(doc.get("name"));
    System.out.println(doc.get("author"));
}
```


### 删除索引

```java
// 文档删除
IndexWriter indexWriter = new IndexWriter(directory, indexWriterConfig);
indexWriter.deleteDocuments(new Term("name", keyword));// 删除查询到的文档
indexWriter.forceMergeDeletes();// 未merge之前可以恢复4.0之前版本
indexWriter.commit();
indexWriter.close();

// 文档恢复 4.0以后不再支持
indexReader.undeleteAll();
```

### 更新索引

Lucene没有提供更新操作，更新实质上是先删除再添加，因为更新索引很消耗性能

```java
indexWriter.updateDocument(new Term("id","1"),document);
```

### 分词

#### 分词器种类

* SimpleAnalyzer
* StandardAnalyzer
* StopAnalyzer（StopwordAnalyzerBase）
* WhitespaceAnalyzer

#### TokenStream

分词器组偏好处理之后得到的一个流，这个流中存储的各种信息，可以通过TokenStream有效的获取到分词单元信息

##### TokenStream存储结构

![TokenStream](/static/img/Lucene/TokenStream.png "TokenStream")

##### Reader处理成TokenStream

![Reader2TokenStream.png](/static/img/Lucene/Reader2TokenStream.png "Reader2TokenStream.png")

* Tokenizer extends TokenStream：将一组数据划分不同的词汇单元，将Reader进行分词操作
* TokenFilter extends TokenStream：对词汇单元单元进行过滤

![Tokenizer.png](/static/img/Lucene/Tokenizer.png "Tokenizer.png")

![TokenFilter.png](/static/img/Lucene/TokenFilter.png "TokenFilter.png")

* Attribute
    - PositionIncrementAttribute 位置增量属性，存储语汇单元之间的距离
    - OffsetAttribute 每个语汇单元的位置偏移量
    - CharTermAttribute 存储每一个语汇单元的信息（分词单元信息）
    - TypeAttribute 使用的分词器的类型信息

```java
public List<String> analyzerTest(String content) throws IOException {
    boolean useSmart = true;
    IKSegmenter ik = new IKSegmenter(new StringReader(content), useSmart);
    Lexeme word;
    List<String> result = new ArrayList<String>();
    while ((word = ik.next()) != null) {
        result.add(word.getLexemeText());
    }
    return result;
}

List<String> strings = indexAndSearchService.analyzerTest("hello,大家好我是一个中国人");
// hello
// 大家好
// 我
// 是
// 一个
// 中国人


public List<String> analyzerDemo(String content) {
    boolean useSmart = true;
    // 构建IK分词器，使用smart分词模式
    Analyzer analyzer = new IKAnalyzer(true);
    // 获取Lucene的TokenStream对象
    TokenStream tokenStream = null;
    List<String> strings = new ArrayList<String>();
    try {
        tokenStream = analyzer.tokenStream("name", new StringReader(content));
        // 获取词元位置属性
        OffsetAttribute offset = tokenStream.addAttribute(OffsetAttribute.class);
        // 获取词元文本属性
        CharTermAttribute term = tokenStream.addAttribute(CharTermAttribute.class);
        // 获取词元文本属性
        TypeAttribute type = tokenStream.addAttribute(TypeAttribute.class);
        // 位置增量
        PositionIncrementAttribute position = tokenStream.addAttribute(PositionIncrementAttribute.class);
        // 重置TokenStream（重置StringReader）
        tokenStream.reset();
        // 迭代获取分词结果
        while (tokenStream.incrementToken()) {
            String word = term.toString();
            System.out.println(position.getPositionIncrement() + " " + offset.startOffset() + " - "
                    + offset.endOffset() + " : " + word + " | "
                    + type.type());
            strings.add(word);
        }
        // 关闭TokenStream（关闭StringReader）
        tokenStream.end(); // Perform end-of-stream operations, e.g. set the final offset.
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        // 释放TokenStream的所有资源
        if (tokenStream != null) {
            try {
                tokenStream.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    return strings;
}

// 0 - 5 : hello | ENGLISH
// 6 - 9 : 大家好 | CN_WORD
// 9 - 10 : 我 | CN_WORD
// 10 - 11 : 是 | CN_CHAR
// 11 - 13 : 一个 | CN_WORD
// 13 - 16 : 中国人 | CN_WORD
// hello
// 大家好
// 我
// 是
// 一个
// 中国人
```

#### 自定义分词

```java
// 组合实现分词器功能
public class MyAnalyzer extends Analyzer {
    private CharArraySet stopWordSet;
    public MyAnalyzer() {
        // 增加默认的英文停用词
        stopWordSet = new CharArraySet(Version.LUCENE_43, StopAnalyzer.ENGLISH_STOP_WORDS_SET.size(), true);
        stopWordSet.addAll(StopAnalyzer.ENGLISH_STOP_WORDS_SET);
    }
    public MyAnalyzer(String[] stopWords) {
        this();
        // 增加自定义停用词
        CharArraySet c = StopFilter.makeStopSet(Version.LUCENE_43, stopWords);
        this.stopWordSet.addAll(c);
    }
    @Override
    protected TokenStreamComponents createComponents(String fieldName, Reader reader) {
        // 通过单词分词
        LetterTokenizer letterTokenizer = new LetterTokenizer(Version.LUCENE_43, reader);
        // 忽略大小写
        LowerCaseFilter lowerCaseFilter = new LowerCaseFilter(Version.LUCENE_43, letterTokenizer);
        // 停用词
        StopFilter stopFilter = new StopFilter(Version.LUCENE_43, lowerCaseFilter, stopWordSet);
        return new TokenStreamComponents(letterTokenizer, stopFilter);
    }
}
```

```java
// 同义词分词器
public class SameWordsFilter extends TokenFilter {
    private PositionIncrementAttribute positionIncrementAttribute;
    private CharTermAttribute charTermAttribute;
    private Stack<String> sameWordStack = new Stack<String>();
    private Map<String, String[]> sameWords = new HashMap<String, String[]>();
    public SameWordsFilter(TokenStream input) {
        super(input);
    }
    public SameWordsFilter(TokenStream input, Map<String, String[]> sameWords) {
        this(input);
        this.sameWords = sameWords;
        // 获取位置信息
        positionIncrementAttribute = this.addAttribute(PositionIncrementAttribute.class);
        // 获取字符信息
        charTermAttribute = this.addAttribute(CharTermAttribute.class);
    }
    @Override
    public final boolean incrementToken() throws IOException {
        if (!sameWordStack.empty()) {
            // 将同义词设置到指定位置
            charTermAttribute.setEmpty();
            charTermAttribute.append(sameWordStack.pop());// 取出同义词
            positionIncrementAttribute.setPositionIncrement(0);
            return true;
        }
        if (this.input.incrementToken()) {// 下一个分词
            // 取出下一同义词
            getSameWords(charTermAttribute.toString());
            return true;
        }
        return false;
    }
    // 获取同义词并存储用于处理
    private boolean getSameWords(String word) {
        String[] sames = sameWords.get(word);
        if (ArrayUtils.isEmpty(sames)) {
            return false;
        }
        for (String same : sames) {
            sameWordStack.push(same);
        }
        return true;

    }
}
```

### 高亮器

用于截取一段文本形成摘要（关键字频率最高的附近多少个字符），并让关键字搞来那个显示（通过前缀和后缀）

```java
public void highlighter(String[] keywords) throws IOException, InvalidTokenOffsetsException, ParseException {
	Analyzer analyzer = new StandardAnalyzer(LUCENE_43);

    QueryParser queryParser = new MultiFieldQueryParser(LUCENE_43, new String[]{"content"}, analyzer);
    Query query = queryParser.parse(keywords[1]);

    Formatter formatter = new SimpleHTMLFormatter("<font color='red'>", "</font>");
    Encoder encoder = new DefaultEncoder();
    Scorer scorer = new QueryScorer(query);
    // init highlighter
    Highlighter highlighter = new Highlighter(formatter, encoder, scorer);
    // 这里指定最大的摘要长度，超过长度会截取
    SimpleFragmenter fragmenter = new SimpleFragmenter(50);
    highlighter.setTextFragmenter(fragmenter);
    // 注意如果没有找到高亮内容会返回空
    String bestFragment = highlighter.getBestFragment(analyzer, "content", "我是中国人");
    System.out.println(bestFragment);
}

// StandardAnalyzer单个分词中文，所以分词结果中分开高亮
// indexAndSearchService.highlighter(new String[]{"name", "中国"});
// 我是<font color='red'>中</font><font color='red'>国</font>人
```

### 数据同步

![sync.png](/static/img/Lucene/sync.png "sync.png")

### 工具

#### Luke索引查看工具

Luke is the GUI tool for introspecting your Lucene / Solr / Elasticsearch index.

<https://github.com/DmitryKey/luke>

#### Tika内容抽取工具

Tika是一个内容抽取的工具集合(a toolkit for text extracting)。它集成了POI, Pdfbox 并且为文本抽取工作提供了一个统一的界面。

<http://tika.apache.org/>

```java
// 直接使用tika
FileInputStream fileInputStream = FileUtils.openInputStream(new File("c:\\Users\\xpress\\Downloads\\Documents\\Java求职宝典2014版.pdf"));
Metadata metadata = new Metadata();

Tika tika = new Tika();
try {
    String string = tika.parseToString(fileInputStream, metadata);
    for (String name : metadata.names()) {
        System.out.println(name + ":" + metadata.get(name));
    }
    System.out.println(string);// 获取的文档内容
} catch (TikaException e) {
    e.printStackTrace();
}

// 使用Parser
AutoDetectParser autoDetectParser = new AutoDetectParser();
int writeLimit = 2000000;
BodyContentHandler bodyContentHandler = new BodyContentHandler(writeLimit);
ParseContext parseContext = new ParseContext();
parseContext.set(AutoDetectParser.class, autoDetectParser);
FileInputStream fileInputStream = FileUtils.openInputStream(new File("Java.pdf"));
try {
    Metadata metadata = new Metadata();
    autoDetectParser.parse(fileInputStream, bodyContentHandler, metadata, parseContext);
    for (String name : metadata.names()) {// 元信息
        System.out.println(name+":"+metadata.get(name));
    }
    System.out.println(bodyContentHandler.toString());// 获取的文档内容
} catch (SAXException e) {
    e.printStackTrace();
} catch (TikaException e) {
    e.printStackTrace();
} finally {
    if (fileInputStream != null) {
        fileInputStream.close();
    }
}
```

## Solr

### 配置

#### solr.home配置

* 在web.xml中

```xml
 <env-entry>
    <env-entry-name>solr/home</env-entry-name>
    <env-entry-type>java.lang.String</env-entry-type>
    <env-entry-value>/home/xpress/Solr/solrhome</env-entry-value>
</env-entry>
```

* 在server.xml中

```xml
<Context path="" docBase="solr" reloadable="false" crossContext="true">
    <Environment name="solr/home" type="java.lang.String" value="/home/xpress/Solr/solrhome" override="true"/>
</Context>
```

#### Schema

managed-schema

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<schema name="dbcore" version="1.6">

    <field name="_version_" type="long" indexed="true" stored="true"/>
    <field name="_root_" type="string" indexed="true" stored="false"/>
    <field name="text" type="text_cn" indexed="true" stored="false" multiValued="true"/>
    <field name="id" type="string" indexed="true" stored="true" required="true" multiValued="false"/>
    <field name="name" type="text_cn" indexed="true" stored="true" />
    <field name="description" type="text_cn" indexed="true" stored="true" />
    <!-- 配合copy，多字段联合查询 -->
    <field name="all" type="text_cn" indexed="true" stored="false" multiValued="true"/>

    <dynamicField name="*_i" type="int" indexed="true" stored="true"/>
    <dynamicField name="random_*" type="random"/>
    <!-- 唯一标识 -->
    <uniqueKey>id</uniqueKey>
    <!-- 多值字段 -->
    <copyField source="name" dest="all"/>
    <copyField source="description" dest="all"/>
    <!-- solr自带中文分词类型 -->
    <fieldType name="text_cn" class="solr.TextField" positionIncrementGap="0">
        <analyzer type="index">
            <tokenizer class="org.apache.lucene.analysis.cn.smart.HMMChineseTokenizerFactory"/>
        </analyzer>
        <analyzer type="query">
            <tokenizer class="org.apache.lucene.analysis.cn.smart.HMMChineseTokenizerFactory"/>
        </analyzer>
    </fieldType>

    <fieldType name="string" class="solr.StrField" sortMissingLast="true"/>
    <fieldType name="int" class="solr.TrieIntField" precisionStep="0" positionIncrementGap="0"/>
    <fieldType name="float" class="solr.TrieFloatField" precisionStep="0" positionIncrementGap="0"/>
    <fieldType name="long" class="solr.TrieLongField" precisionStep="0" positionIncrementGap="0"/>
    <fieldType name="double" class="solr.TrieDoubleField" precisionStep="0" positionIncrementGap="0"/>
    <fieldType name="tdate" class="solr.TrieDateField" precisionStep="6" positionIncrementGap="0"/>
    <fieldType name="random" class="solr.RandomSortField" indexed="true"/>
    <fieldType name="point" class="solr.PointType" dimension="2" subFieldSuffix="_d"/>
    <!-- A specialized field for geospatial search. If indexed, this fieldType must not be multivalued. -->
    <fieldType name="location" class="solr.LatLonType" subFieldSuffix="_coordinate"/>
    <!-- An alternative geospatial field type new to Solr 4.  It supports multiValued and polygon shapes.
      For more information about this and other Spatial fields new to Solr 4, see:
      http://wiki.apache.org/solr/SolrAdaptersForLuceneSpatial4
    -->
    <fieldType name="location_rpt" class="solr.SpatialRecursivePrefixTreeFieldType"
               geo="true" distErrPct="0.025" maxDistErr="0.001" distanceUnits="kilometers"/>
    <fieldType name="currency" class="solr.CurrencyField" precisionStep="8" defaultCurrency="USD" currencyConfig="currency.xml"/>
</schema>
```

### DIH

dataimporthandler

solrconfig.xml

```xml
<requestHandler name="/dataimport" class="solr.DataImportHandler">
    <lst name="defaults">
        <str name="config">db-data-config.xml</str>
    </lst>
</requestHandler>
```

db-data-config.xml

```xml
<dataConfig>
    <dataSource driver="com.mysql.jdbc.Driver" url="jdbc:mysql://192.168.94.130:3306/mydb" user="username" password="password"/>
    <document>
        <entity name="item" query="select * from book"
                deltaQuery="select id from item where last_modified > '${dataimporter.last_index_time}'">
            <field column="id" name="id"/>
            <field column="name" name="name"/>
            <field column="description" name="description"/>
        </entity>
    </document>
</dataConfig>
```

### SolrJ

```java
public class SolrJTest {
    private HttpSolrClient httpSolrClient;
    @Before
    public void before() {
        String url = "http://localhost:8081/dbcore";
        HttpSolrClient.Builder builder = new HttpSolrClient.Builder(url);
        httpSolrClient = builder.build();
    }
    @Test
    public void deleteDocument() throws Exception{
        httpSolrClient.deleteByQuery("id:*");
        httpSolrClient.commit();
    }
    @Test
    public void addDocument() throws Exception{
        SolrInputDocument document = new SolrInputDocument();
        document.addField("id", "1");
        document.addField("name", "bob");
        document.addField("description", "搜索引擎（Search Engine）");
        httpSolrClient.add(document);
        httpSolrClient.commit();
    }
    @Test
    public void query() throws Exception{
        SolrQuery solrParams = new SolrQuery();
        // 查询条件
        solrParams.set("q","*是*");
        solrParams.set("df","all");//默认字段
        // solrParams.set("q","description:*是*");
        // 分页
        solrParams.setStart(3);// 起始行数
        solrParams.setRows(3);// 查询条数
        // 高亮
        solrParams.addHighlightField("description");
        solrParams.setHighlight(true);
        solrParams.setHighlightSnippets(1);
        solrParams.setHighlightSimplePre("<b>");
        solrParams.setHighlightSimplePost("</b>");

        QueryResponse queryResponse = httpSolrClient.query(solrParams);
        System.out.println(queryResponse.getResults().getNumFound());// 查询到的总结果数
        List<MyBean> beans = queryResponse.getBeans(MyBean.class);
        for (MyBean bean : beans) {
            // 获取高亮字段
            List<String> description = queryResponse.getHighlighting().get(bean.getId()).get("description");
            if (description.size() > 0) {
                bean.setDescription(description.get(0));
            }
            System.out.println(bean);
        }

    }
    @After
    public void after() throws IOException {
        httpSolrClient.close();
    }
}

// 使用注解和实体类
public class MyBean {
    public MyBean(String id, String name, String description) {
        this.id = id;
        this.name = name;
        this.description = description;
    }
    @Field
    private String id;
    @Field
    private String name;
    @Field
    private String description;
    // geter seter...
}

List<MyBean> myBeans = new ArrayList<MyBean>();
myBeans.add(new MyBean("1", "smith", "大叫好我是smith"));
myBeans.add(new MyBean("2", "bob", "大叫好我是bob"));
myBeans.add(new MyBean("3", "tom", "大叫好我是tom"));
try {
    httpSolrClient.addBeans(myBeans);
    httpSolrClient.commit();
} catch (SolrServerException e) {
    e.printStackTrace();
} catch (IOException e) {
    e.printStackTrace();
}
```

### 应用

#### 搜索

```
q=price:[10 TO 60] AND iPhone
```

#### 权重

```
defType=edismax
qf=prod_name^1.0+sale_tag^5.0+summary^0.5
```

#### facet分面

**分面字段**

```
facet.field=brand_id
```

分面统计和fq协同

```
fq={!tag=property_1}property_id:1
// 通过tag控制是否统计fq过滤的信息
facet.field={!ex=property_1}property_id
```

**价格区间**

```
facet.range=price&f.price.facet.range.start=0.0&f.price.facet.range.end=1000.0&f.price.facet.range.gap=100&f.price.facet.sort=index
```

**所属类目树统计**

```
facet.pivot=...id
```

### Solr优化

* 索引文件的总体大小
* 记录条数
* 倒排链条的长短

------

*Lucene教程*