---
title: lucene使用小结
date: 2016-04-16 16:32:04
tags:
---

## 简介
Lucene是一套用于全文检索和搜寻的开源程式库。包括Elasticsearch和Solr等开源搜索引擎都基于Lucene实现。

## 原理
简单来说，Lucene就是把很多文本，切割成一个个单词，然后对于这些单词建立索引，索引里包含了单词所在文本的位置。当需要搜索某个单词的时候，可以迅速找到包含这个单词的文本。



Lucene API.
1. org.apache.lucene.analysis Analyzer用于解析文本，把文本拆成一个个单词。
1. org.apache.lucene.store    Directory为Lucene索引存储服务，有基于内存，文件或是数据库的实现。
1. org.apache.lucene.index    提供索引的查询和维护，包含IndexReader和IndexWriter，负责读写index。
1. org.apache.lucene.search   IndexSearcher根据index做查询。

概念:
1. Document。 包含多个field。比如一篇文章包含有标题，摘要，作者，全文等属性。
1. Field。    fieldType可以控制这个field是否加入索引，是否需要存储，是否需要分词。
1. Term。     一个term就是一个单词。

## 使用

### 创建Maven工程
导入Lucene的依赖包。

```xml

	<properties>
		<lucence.version>6.0.0</lucence.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.apache.lucene</groupId>
			<artifactId>lucene-core</artifactId>
			<version>${lucence.version}</version>
		</dependency>
		<dependency>
			<groupId>org.apache.lucene</groupId>
			<artifactId>lucene-analyzers-common</artifactId>
			<version>${lucence.version}</version>
		</dependency>
		<dependency>
			<groupId>org.apache.lucene</groupId>
			<artifactId>lucene-queryparser</artifactId>
			<version>${lucence.version}</version>
		</dependency>
	</dependencies>

```

### 索引
把文本加入到Lucene索引中。
```java
		FieldType myFieldType = new FieldType(TextField.TYPE_STORED)
        Directory dir = new RAMDirectory();
    	Analyzer analyzer = new StandardAnalyzer();
    	IndexWriterConfig iwc = new IndexWriterConfig(analyzer);
    	try(IndexWriter writer = new IndexWriter(dir, iwc)){
	    	Document doc = new Document();
   			doc.add(new Field("content","Lucene is a Java full-text search engine. Lucene is not a complete application, but rather a code library and API that can easily be used to add search capabilities to applications.", myFieldType));
	    	writer.addDocument(doc);
	    	Document doc = new Document();
   			doc.add(new Field("content","Apache Lucene is an open source project available for free download.", myFieldType));
	    	writer.addDocument(doc);	    	
    	} catch (IOException e) {
			e.printStackTrace();
		}

```
### 检索
通过IndexSearcher查询，下面的例子中会搜索包含java和lucene的文档。
```java
        Directory dir = new RAMDirectory();
    	Analyzer analyzer = new StandardAnalyzer();
    	try(IndexReader  reader = DirectoryReader.open(dir)){
	    	IndexSearcher searcher = new IndexSearcher(reader);
	    	QueryParser parser = new QueryParser("content", analyzer);
	    	Query query = parser.parse("Lucene Java");
	    	TopDocs results =searcher.search(query, 100);
	    	ScoreDoc[] hits = results.scoreDocs;
	    	for(ScoreDoc hit: hits){
	    		Document doc = searcher.doc(hit.doc);
	    		System.out.println(doc);
	    	}
    	} catch (IOException e) {
			e.printStackTrace();
		} catch (ParseException e) {
			e.printStackTrace();
		}

```

### 统计词频
通过index可以知道某个term的使用频率。
```java

        Directory dir = new RAMDirectory();
    	Analyzer analyzer = new StandardAnalyzer();
    	try(IndexReader  reader = DirectoryReader.open(dir)){
    		Term term = new Term("content","lucene")
	    	System.out.println(term.toString()+"  "+reader.totalTermFreq(term)+"  "+reader.docFreq(term)	);
    	} catch (IOException e) {
			e.printStackTrace();
		} catch (ParseException e) {
			e.printStackTrace();
		}

```
也可以遍历所有term，不过需要fieldType支持StoreTermVectors。

```java

   for(int i=0;i<reader.maxDoc();i++){
	   Terms doc = reader.getTermVector(i, key);
	   if(doc == null){
		   continue;
	   }
	   TermsEnum te = doc.iterator();
	   BytesRef br = null;
	   while((br=te.next()) != null){
		   Term term = new Term("content",br);
		   System.out.println(term.toString()+"  "+reader.totalTermFreq(term)+"  "+reader.docFreq(term)	);
		   
	   }
	   
   }

```

## 分词分析器
Lucene提供了各种Analyzer针对于不同的语言和文本。比如“Apache Lucene is an open source project”。这其中is和a就是没啥意义的词，一般不会用于索引，所以分析器会在分词过程中过滤掉。

各种分析器本质上依赖其中包含的tokenstream，一个分析器可能包含多个stream，原始文本经过一个个stream按顺序处理，直到得到最终结果。
Source-->Stream1-->Stream2-->...StreamN-->Result

主要有两种tokenstream:
1. Tokenizer。把文本分割成一个个单词，比如英语可以根据空格分割。一般Tokennizer是第一个tokenSteam。
1. TokenFilter。 把分割出来的单词做过滤或修改，比如前面说的一些不需要索引的单词。

比如StandardAnalyzer通过createComponents创建了下面集中tokenizer和filter。如果我们自己写一个analyzer也可以重写这个方法来定制tokenStream。
```java

  protected TokenStreamComponents createComponents(final String fieldName) {
    final StandardTokenizer src = new StandardTokenizer();
    src.setMaxTokenLength(maxTokenLength);
    TokenStream tok = new StandardFilter(src);
    tok = new LowerCaseFilter(tok);
    tok = new StopFilter(tok, stopwords);
    return new TokenStreamComponents(src, tok) {
      @Override
      protected void setReader(final Reader reader) {
        src.setMaxTokenLength(StandardAnalyzer.this.maxTokenLength);
        super.setReader(reader);
      }
    };
  }

```

在tokenStream的处理过程中，可以通过Attribute来知道前面一个tokenStream的状态。
