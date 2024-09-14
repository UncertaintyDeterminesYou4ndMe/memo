# ES
# 引入依赖包

```xml
<!-- es -->
<dependency>
  <groupId>org.elasticsearch</groupId>
  <artifactId>elasticsearch</artifactId>
  <version>7.10.1</version>
</dependency>
<dependency>
  <groupId>org.elasticsearch.client</groupId>
  <artifactId>elasticsearch-rest-client</artifactId>
  <version>7.10.1</version>
</dependency>
<dependency>
  <groupId>org.elasticsearch.client</groupId>
  <artifactId>elasticsearch-rest-high-level-client</artifactId>
  <version>7.10.1</version>
</dependency>
```



# ES数据源配置

```properties
# es配置
elasticsearch.config.host=test.middleware.shuli.com
elasticsearch.config.port=9200
elasticsearch.config.scheme=http
elasticsearch.config.username=
elasticsearch.config.password=
elasticsearch.config.maxConnTotal=100
elasticsearch.config.maxConnPerRoute=100
```



# ES配置工具类

```java
package com.shulidata.marketing.common.integration.elasticsearch.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;

/**
 * es配置属性
 *
 * @author kaifang.z on 2023/4/10
 */
@Data
@ConfigurationProperties(prefix = "elasticsearch.config")
public class ElasticsearchProperties {

    /**
     * 域名
     */
    private String host;
    /**
     * 端口
     */
    private int port;
    /**
     * 链接方式
     */
    private String scheme;
    /**
     * 用户
     */
    private String username;
    /**
     * 密码
     */
    private String password;
    /**
     * 最大连接数
     */
    private int maxConnTotal;
    /**
     * 最大路由连接数
     */
    private int maxConnPerRoute;

}


```

```java

import org.apache.http.HttpHost;
import org.apache.http.auth.AuthScope;
import org.apache.http.auth.UsernamePasswordCredentials;
import org.apache.http.client.CredentialsProvider;
import org.apache.http.impl.client.BasicCredentialsProvider;
import org.apache.http.impl.nio.client.HttpAsyncClientBuilder;
import org.elasticsearch.client.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * es配置定义
 *
 * @author kaifang.z on 2023/4/10
 */
@Configuration
@EnableConfigurationProperties(ElasticsearchProperties.class)
public class ElasticsearchConfig {

    @Autowired
    private ElasticsearchProperties properties;

    @Bean
    public RestHighLevelClient restHighLevelClient() {
        final CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
        credentialsProvider.setCredentials(AuthScope.ANY, new UsernamePasswordCredentials(properties.getUsername(), properties.getPassword()));
        RestHighLevelClient restHighLevelClient = new RestHighLevelClient(
            RestClient.builder(new HttpHost(properties.getHost(), properties.getPort(), properties.getScheme())).setHttpClientConfigCallback(new RestClientBuilder.HttpClientConfigCallback() {
                public HttpAsyncClientBuilder customizeHttpClient(HttpAsyncClientBuilder httpClientBuilder) {
                    httpClientBuilder.setMaxConnTotal(properties.getMaxConnTotal());
                    httpClientBuilder.setMaxConnPerRoute(properties.getMaxConnPerRoute());
                    return httpClientBuilder.setDefaultCredentialsProvider(credentialsProvider);
                }
            })
        );
        return restHighLevelClient;
    }
}

```



```java

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.elasticsearch.action.DocWriteResponse;
import org.elasticsearch.action.admin.indices.delete.DeleteIndexRequest;
import org.elasticsearch.action.bulk.BulkRequest;
import org.elasticsearch.action.bulk.BulkResponse;
import org.elasticsearch.action.delete.DeleteRequest;
import org.elasticsearch.action.delete.DeleteResponse;
import org.elasticsearch.action.get.GetRequest;
import org.elasticsearch.action.get.GetResponse;
import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.action.index.IndexResponse;
import org.elasticsearch.action.search.SearchRequest;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.action.support.WriteRequest;
import org.elasticsearch.action.support.master.AcknowledgedResponse;
import org.elasticsearch.action.update.UpdateRequest;
import org.elasticsearch.action.update.UpdateResponse;
import org.elasticsearch.client.RequestOptions;
import org.elasticsearch.client.RestHighLevelClient;
import org.elasticsearch.client.indices.CreateIndexRequest;
import org.elasticsearch.client.indices.CreateIndexResponse;
import org.elasticsearch.client.indices.GetIndexRequest;
import org.elasticsearch.common.unit.TimeValue;
import org.elasticsearch.common.xcontent.XContentBuilder;
import org.elasticsearch.common.xcontent.XContentFactory;
import org.elasticsearch.common.xcontent.XContentType;
import org.elasticsearch.index.query.BoolQueryBuilder;
import org.elasticsearch.search.SearchHit;
import org.elasticsearch.search.builder.SearchSourceBuilder;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.ArrayList;
import java.util.List;
import java.util.Objects;

/**
 * es工具类
 */
@Component
@Slf4j
public class ElasticsearchUtils {

    /**
     * 超时时间 10s
     */
    public static final TimeValue TIME_VALUE_SECONDS = TimeValue.timeValueSeconds(10);
    /**
     * 批量最大处理数
     */
    private static final int MAX_BATCH_SIZE = 10000;

    @Autowired
    private RestHighLevelClient restHighLevelClient;


    /**
     * 创建索引,若索引不存在且创建成功,返回true,若同名索引已存在,返回false
     *
     * @param indexName 索引名称
     * @return
     */
    public boolean createIndex(String indexName, String mappingJson) {
        LoggerUtil.info(log, "es新增索引，indexName={}", indexName);
        CreateIndexRequest request = new CreateIndexRequest(indexName);
        try {
            if(StringUtils.isNotBlank(mappingJson)){
               XContentBuilder builder = XContentFactory.jsonBuilder().map(JSONObject.parseObject(mappingJson));
               request.mapping(builder);
               }
               CreateIndexResponse response = restHighLevelClient.indices().create(request, RequestOptions.DEFAULT);
               return response.isAcknowledged();
               } catch (Exception e) {
               LoggerUtil.error(log, "es新增索引异常", e);
               throw new CommonException(ErrorCode.ERR_ES_QUERY);
               }
               }

               /**
               * 判断索引是否存在
               *
               * @param indexName 索引名称
               * @return
               */
               public boolean exitsIndex(String indexName) {
               GetIndexRequest request = new GetIndexRequest(indexName);
               try {
               return restHighLevelClient.indices().exists(request, RequestOptions.DEFAULT);
               } catch (Exception e) {
               LoggerUtil.error(log, "es查询索引是否存在异常", e);
               throw new CommonException(ErrorCode.ERR_ES_QUERY);
               }
               }

               /**
               * 删除索引
               *
               * @param indexName 索引名称
               * @return
               */
               public boolean deleteIndex(String indexName) {
               LoggerUtil.info(log, "es删除索引，indexName={}", indexName);
               DeleteIndexRequest request = new DeleteIndexRequest(indexName);
               try {
               AcknowledgedResponse response = restHighLevelClient.indices().delete(request, RequestOptions.DEFAULT);
               return response.isAcknowledged();
               } catch (Exception e) {
               LoggerUtil.error(log, "es删除索引异常", e);
               throw new CommonException(ErrorCode.ERR_ES_QUERY);
               }
               }


               /**
               * 新增/修改文档信息
               *
               * @param indexName 索引
               * @param id        文档id
               * @param data      数据
               * @return
               */
               public boolean createOrUpdateDoc(String indexName, String id, Object data) {
               LoggerUtil.info(log, "es新增/更新文档，indexName={}，id={}", indexName, id);
               IndexRequest request = new IndexRequest(indexName).id(id).timeout(TIME_VALUE_SECONDS);
               request.source(JSON.toJSONString(data), XContentType.JSON);
               try {
               IndexResponse response = restHighLevelClient.index(request, RequestOptions.DEFAULT);
               return response.getResult().equals(DocWriteResponse.Result.CREATED);
               } catch (Exception e) {
               LoggerUtil.error(log, "es新增文档异常", e);
               throw new CommonException(ErrorCode.ERR_ES_QUERY);
               }
               }


               /**
               * 更新文档信息
               *
               * @param indexName 索引
               * @param id        文档id
               * @param data      数据
               * @return
               */
               public boolean updateDoc(String indexName, String id, Object data) {
               LoggerUtil.info(log, "es更新文档，indexName={}，id={}", indexName, id);
               UpdateRequest request = new UpdateRequest(indexName, id);
               request.doc(JSON.toJSONString(data), XContentType.JSON);
               request.timeout(TIME_VALUE_SECONDS);
               request.setRefreshPolicy(WriteRequest.RefreshPolicy.WAIT_UNTIL);
               try {
               UpdateResponse response = restHighLevelClient.update(request, RequestOptions.DEFAULT);
               return response.getGetResult().equals(DocWriteResponse.Result.UPDATED);
               } catch (Exception e) {
               LoggerUtil.error(log, "es更新文档异常", e);
               throw new CommonException(ErrorCode.ERR_ES_QUERY);
               }
               }

               /**
               * 批量新增数据
               * 建议几百条数据执行一次
               *
               * @param dataList
               * @return
               */
               public boolean batchCreateDoc(List<? extends ESBaseData> dataList) {
               if (dataList.size() > MAX_BATCH_SIZE) {
               LoggerUtil.error(log, "es批量新增文档失败，数据量不能大于1W条", dataList.size());
               throw new CommonException(ErrorCode.ERR_ES_QUERY);
               }
               try {
               BulkRequest request = new BulkRequest();
               dataList.forEach(data -> {
               String source = JSON.toJSONString(data);
               request.add(new IndexRequest(data.getIndexName()).id(data.id).source(source, XContentType.JSON));
               });
               BulkResponse response = restHighLevelClient.bulk(request, RequestOptions.DEFAULT);
               return !response.hasFailures();
               } catch (Exception e) {
               LoggerUtil.error(log, "es批量新增文档异常", e);
               throw new CommonException(ErrorCode.ERR_ES_QUERY);
               }
               }

               /**
               * 通过id删除文档
               *
               * @param indexName
               * @param id
               * @return
               */
               public boolean deleteDocById(String indexName, String id) {
               LoggerUtil.info(log, "es根据ID删除文档, indexName={},id={}", indexName, id);
               DeleteRequest deleteRequest = new DeleteRequest(indexName, id);
               try {
               DeleteResponse response = restHighLevelClient.delete(deleteRequest, RequestOptions.DEFAULT);
               return response.getResult().equals(DocWriteResponse.Result.DELETED);
               } catch (Exception e) {
               LoggerUtil.error(log, "es根据ID删除文档异常", e);
               throw new CommonException(ErrorCode.ERR_ES_QUERY);
               }
               }

               /**
               * 批量刪除文档
               *
               * @param indexName 索引名称
               * @param idList    需要批量删除的id集合
               * @return
               */
               public boolean batchDeleteDoc(String indexName, List<String> idList) {
               LoggerUtil.info(log, "es批量删除文档, indexName={},idList={}", indexName, idList);
               BulkRequest request = new BulkRequest();
               idList.forEach(id -> request.add(new DeleteRequest().index(indexName).id(id)));
               try {
               BulkResponse response = restHighLevelClient.bulk(request, RequestOptions.DEFAULT);
               return !response.hasFailures();
               } catch (Exception e) {
               LoggerUtil.error(log, "es批量删除文档异常", e);
               throw new CommonException(ErrorCode.ERR_ES_QUERY);
               }
               }

               /**
               * 根据id查询文档
               */
               public <T> T selectDocById(String indexName, String id, Class<T> c) {
               GetRequest indexRequest = new GetRequest(indexName, id);
               try {
               GetResponse response = restHighLevelClient.get(indexRequest, RequestOptions.DEFAULT);
               return JSONObject.parseObject(response.getSourceAsString(), c);
               } catch (Exception e) {
               LoggerUtil.error(log, "es根据ID查询文档异常", e);
               throw new CommonException(ErrorCode.ERR_ES_QUERY);
               }
               }

               /**
               * 判断文档是否存在
               *
               * @param indexName 索引
               * @param id        文档id
               * @return
               */
               public boolean existsDoc(String indexName, String id) {
               GetRequest request = new GetRequest(indexName, id);
               try {
               GetResponse response = restHighLevelClient.get(request, RequestOptions.DEFAULT);
               return response.isExists();
               } catch (Exception e) {
               LoggerUtil.error(log, "es判断文档是否存在异常", e);
               throw new CommonException(ErrorCode.ERR_ES_QUERY);
               }
               }

               /**
               * 条件查询
               *
               * @param indexName
               * @param queryBuilder 条件
               * @param c            返回对象类型
               * @return
               */
               public <T> List<T> selectDocList(String indexName, BoolQueryBuilder queryBuilder, Class<T> c) {
               SearchRequest searchRequest = new SearchRequest();
               searchRequest.indices(indexName);
               SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
               sourceBuilder.trackTotalHits(true);
               sourceBuilder.query(queryBuilder);
               searchRequest.source(sourceBuilder);
               try {
               SearchResponse searchResponse = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
               SearchHit[] hits = searchResponse.getHits().getHits();
               List<T> list = new ArrayList<>(hits.length);
               for (SearchHit hit : hits) {
               list.add(JSONObject.parseObject(hit.getSourceAsString(), c));
               }
               return list;
               } catch (Exception e) {
               LoggerUtil.error(log, "es根据条件查询文档列表异常", e);
               throw new CommonException(ErrorCode.ERR_ES_QUERY);
               }
               }

               /**
               * 条件查询
               *
               * @param indexName
               * @param queryBuilder 条件
               * @return
               */
               public ESPage<String> selectIdListPage(String indexName, BoolQueryBuilder queryBuilder, Integer pageNum,
               Integer pageSize) {
               SearchRequest searchRequest = new SearchRequest();
               searchRequest.indices(indexName);
               SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
               sourceBuilder.trackTotalHits(true);
               sourceBuilder.query(queryBuilder);
               if (Objects.nonNull(pageNum) && Objects.nonNull(pageSize)) {
               sourceBuilder.from((pageNum - 1) * pageSize).size(pageSize);
               }
               searchRequest.source(sourceBuilder);
               try {
               SearchResponse searchResponse = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
               SearchHit[] hits = searchResponse.getHits().getHits();
               Integer total = new Long(searchResponse.getHits().getTotalHits().value).intValue();
               List<String> list = new ArrayList<>(hits.length);
               for (SearchHit hit : hits) {
               list.add(hit.getId());
               }
               return new ESPage(pageNum, pageSize, total, list);
               } catch (Exception e) {
               LoggerUtil.error(log, "es分页查询文档ID异常", e);
               throw new CommonException(ErrorCode.ERR_ES_QUERY);
               }
               }

               /**
               * 分页查询
               *
               * @param indexName
               * @param pageNum
               * @param pageSize
               * @param queryBuilder
               * @param c
               * @param <T>
               * @return
               */
               public <T> ESPage<T> selectDocPage(String indexName, Integer pageNum, Integer pageSize,
               BoolQueryBuilder queryBuilder, Class<T> c) {
               SearchRequest searchRequest = new SearchRequest();
               searchRequest.indices(indexName);
               SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
               sourceBuilder.trackTotalHits(true);
               sourceBuilder.query(queryBuilder);
               sourceBuilder.from((pageNum - 1) * pageSize).size(pageSize);
               searchRequest.source(sourceBuilder);
               try {
               SearchResponse response = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
               SearchHit[] hits = response.getHits().getHits();
               Integer total = new Long(response.getHits().getTotalHits().value).intValue();
               List<T> list = new ArrayList<>(hits.length);
               for (SearchHit hit : hits) {
               list.add(JSONObject.parseObject(hit.getSourceAsString(), c));
               }
               return new ESPage(pageNum, pageSize, total, list);
               } catch (Exception e) {
               LoggerUtil.error(log, "es分页查询文档列表异常", e);
               throw new CommonException(ErrorCode.ERR_ES_QUERY);
               }
               }

               }


```
## 二十七、Elasticsearch 

### 一、常用查询

```java
package com.wenbronk.javaes;

import java.net.InetSocketAddress;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.Map.Entry;

import org.elasticsearch.action.ListenableActionFuture;
import org.elasticsearch.action.get.GetRequestBuilder;
import org.elasticsearch.action.get.GetResponse;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.action.search.SearchType;
import org.elasticsearch.client.transport.TransportClient;
import org.elasticsearch.common.settings.Settings;
import org.elasticsearch.common.text.Text;
import org.elasticsearch.common.transport.InetSocketTransportAddress;
import org.elasticsearch.common.unit.TimeValue;
import org.elasticsearch.index.query.IndicesQueryBuilder;
import org.elasticsearch.index.query.NestedQueryBuilder;
import org.elasticsearch.index.query.QueryBuilder;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.index.query.QueryStringQueryBuilder;
import org.elasticsearch.index.query.RangeQueryBuilder;
import org.elasticsearch.index.query.SpanFirstQueryBuilder;
import org.elasticsearch.index.query.WildcardQueryBuilder;
import org.elasticsearch.search.SearchHit;
import org.elasticsearch.search.SearchHits;
import org.junit.Before;
import org.junit.Test;

/**
 * java操作查询api
 * @author 231
 *
 */
public class JavaESQuery {
    
    private TransportClient client;
    
    @Before
    public void testBefore() {
        Settings settings = Settings.settingsBuilder().put("cluster.name", "wenbronk_escluster").build();
        client = TransportClient.builder().settings(settings).build()
                 .addTransportAddress(new InetSocketTransportAddress(new InetSocketAddress("192.168.50.37", 9300)));
        System.out.println("success to connect escluster");
    }

    /**
     * 使用get查询
     */
    @Test
    public void testGet() {
        GetRequestBuilder requestBuilder = client.prepareGet("twitter", "tweet", "1");
        GetResponse response = requestBuilder.execute().actionGet();
        GetResponse getResponse = requestBuilder.get();
        ListenableActionFuture<GetResponse> execute = requestBuilder.execute();
        System.out.println(response.getSourceAsString());
    }
    
    /**
     * 使用QueryBuilder
     * termQuery("key", obj) 完全匹配
     * termsQuery("key", obj1, obj2..)   一次匹配多个值
     * matchQuery("key", Obj) 单个匹配, field不支持通配符, 前缀具高级特性
     * multiMatchQuery("text", "field1", "field2"..);  匹配多个字段, field有通配符忒行
     * matchAllQuery();         匹配所有文件
     */
    @Test
    public void testQueryBuilder() {
//        QueryBuilder queryBuilder = QueryBuilders.termQuery("user", "kimchy");
　　　　　　QueryBUilder queryBuilder = QueryBuilders.termQuery("user", "kimchy", "wenbronk", "vini");
        QueryBuilders.termsQuery("user", new ArrayList<String>().add("kimchy"));
//        QueryBuilder queryBuilder = QueryBuilders.matchQuery("user", "kimchy");
//        QueryBuilder queryBuilder = QueryBuilders.multiMatchQuery("kimchy", "user", "message", "gender");
        QueryBuilder queryBuilder = QueryBuilders.matchAllQuery();
        searchFunction(queryBuilder);
        
    }
    
    /**
     * 组合查询
     * must(QueryBuilders) :   AND
     * mustNot(QueryBuilders): NOT
     * should:                  : OR
     */
    @Test
    public void testQueryBuilder2() {
        QueryBuilder queryBuilder = QueryBuilders.boolQuery()
            .must(QueryBuilders.termQuery("user", "kimchy"))
            .mustNot(QueryBuilders.termQuery("message", "nihao"))
            .should(QueryBuilders.termQuery("gender", "male"));
        searchFunction(queryBuilder);
    }
    
    /**
     * 只查询一个id的
     * QueryBuilders.idsQuery(String...type).ids(Collection<String> ids)
     */
    @Test
    public void testIdsQuery() {
        QueryBuilder queryBuilder = QueryBuilders.idsQuery().ids("1");
        searchFunction(queryBuilder);
    }
    
    /**
     * 包裹查询, 高于设定分数, 不计算相关性
     */
    @Test
    public void testConstantScoreQuery() {
        QueryBuilder queryBuilder = QueryBuilders.constantScoreQuery(QueryBuilders.termQuery("name", "kimchy")).boost(2.0f);
        searchFunction(queryBuilder);
        // 过滤查询
//        QueryBuilders.constantScoreQuery(FilterBuilders.termQuery("name", "kimchy")).boost(2.0f);
        
    }
    
    /**
     * disMax查询
     * 对子查询的结果做union, score沿用子查询score的最大值, 
     * 广泛用于muti-field查询
     */
    @Test
    public void testDisMaxQuery() {
        QueryBuilder queryBuilder = QueryBuilders.disMaxQuery()
            .add(QueryBuilders.termQuery("user", "kimch"))  // 查询条件
            .add(QueryBuilders.termQuery("message", "hello"))
            .boost(1.3f)
            .tieBreaker(0.7f);
        searchFunction(queryBuilder);
    }
    
    /**
     * 模糊查询
     * 不能用通配符, 找到相似的
     */
    @Test
    public void testFuzzyQuery() {
        QueryBuilder queryBuilder = QueryBuilders.fuzzyQuery("user", "kimch");
        searchFunction(queryBuilder);
    }
    
    /**
     * 父或子的文档查询
     */
    @Test
    public void testChildQuery() {
        QueryBuilder queryBuilder = QueryBuilders.hasChildQuery("sonDoc", QueryBuilders.termQuery("name", "vini"));
        searchFunction(queryBuilder);
    }
    
    /**
     * moreLikeThisQuery: 实现基于内容推荐, 支持实现一句话相似文章查询
     * {   
        "more_like_this" : {   
        "fields" : ["title", "content"],   // 要匹配的字段, 不填默认_all
        "like_text" : "text like this one",   // 匹配的文本
        }   
    }     
    
    percent_terms_to_match：匹配项（term）的百分比，默认是0.3

    min_term_freq：一篇文档中一个词语至少出现次数，小于这个值的词将被忽略，默认是2
    
    max_query_terms：一条查询语句中允许最多查询词语的个数，默认是25
    
    stop_words：设置停止词，匹配时会忽略停止词
    
    min_doc_freq：一个词语最少在多少篇文档中出现，小于这个值的词会将被忽略，默认是无限制
    
    max_doc_freq：一个词语最多在多少篇文档中出现，大于这个值的词会将被忽略，默认是无限制
    
    min_word_len：最小的词语长度，默认是0
    
    max_word_len：最多的词语长度，默认无限制
    
    boost_terms：设置词语权重，默认是1
    
    boost：设置查询权重，默认是1
    
    analyzer：设置使用的分词器，默认是使用该字段指定的分词器
     */
    @Test
    public void testMoreLikeThisQuery() {
        QueryBuilder queryBuilder = QueryBuilders.moreLikeThisQuery("user")
                            .like("kimchy");
//                            .minTermFreq(1)         //最少出现的次数
//                            .maxQueryTerms(12);        // 最多允许查询的词语
        searchFunction(queryBuilder);
    }
    
    /**
     * 前缀查询
     */
    @Test
    public void testPrefixQuery() {
        QueryBuilder queryBuilder = QueryBuilders.matchQuery("user", "kimchy");
        searchFunction(queryBuilder);
    }
    
    /**
     * 查询解析查询字符串
     */
    @Test
    public void testQueryString() {
        QueryBuilder queryBuilder = QueryBuilders.queryStringQuery("+kimchy");
        searchFunction(queryBuilder);
    }
    
    /**
     * 范围内查询
     */
    public void testRangeQuery() {
        QueryBuilder queryBuilder = QueryBuilders.rangeQuery("user")
            .from("kimchy")
            .to("wenbronk")
            .includeLower(true)     // 包含上界
            .includeUpper(true);      // 包含下届
        searchFunction(queryBuilder);
    }
    
    /**
     * 跨度查询
     */
    @Test
    public void testSpanQueries() {
         QueryBuilder queryBuilder1 = QueryBuilders.spanFirstQuery(QueryBuilders.spanTermQuery("name", "葫芦580娃"), 30000);     // Max查询范围的结束位置  
      
         QueryBuilder queryBuilder2 = QueryBuilders.spanNearQuery()  
                .clause(QueryBuilders.spanTermQuery("name", "葫芦580娃")) // Span Term Queries  
                .clause(QueryBuilders.spanTermQuery("name", "葫芦3812娃"))  
                .clause(QueryBuilders.spanTermQuery("name", "葫芦7139娃"))  
                .slop(30000)                                               // Slop factor  
                .inOrder(false)  
                .collectPayloads(false);  
  
        // Span Not
         QueryBuilder queryBuilder3 = QueryBuilders.spanNotQuery()  
                .include(QueryBuilders.spanTermQuery("name", "葫芦580娃"))  
                .exclude(QueryBuilders.spanTermQuery("home", "山西省太原市2552街道"));  
  
        // Span Or   
         QueryBuilder queryBuilder4 = QueryBuilders.spanOrQuery()  
                .clause(QueryBuilders.spanTermQuery("name", "葫芦580娃"))  
                .clause(QueryBuilders.spanTermQuery("name", "葫芦3812娃"))  
                .clause(QueryBuilders.spanTermQuery("name", "葫芦7139娃"));  
  
        // Span Term  
         QueryBuilder queryBuilder5 = QueryBuilders.spanTermQuery("name", "葫芦580娃");  
    }
    
    /**
     * 测试子查询
     */
    @Test
    public void testTopChildrenQuery() {
        QueryBuilders.hasChildQuery("tweet", 
                QueryBuilders.termQuery("user", "kimchy"))
            .scoreMode("max");
    }
    
    /**
     * 通配符查询, 支持 * 
     * 匹配任何字符序列, 包括空
     * 避免* 开始, 会检索大量内容造成效率缓慢
     */
    @Test
    public void testWildCardQuery() {
        QueryBuilder queryBuilder = QueryBuilders.wildcardQuery("user", "ki*hy");
        searchFunction(queryBuilder);
    }
    
    /**
     * 嵌套查询, 内嵌文档查询
     */
    @Test
    public void testNestedQuery() {
        QueryBuilder queryBuilder = QueryBuilders.nestedQuery("location", 
                QueryBuilders.boolQuery()
                    .must(QueryBuilders.matchQuery("location.lat", 0.962590433140581))
                    .must(QueryBuilders.rangeQuery("location.lon").lt(36.0000).gt(0.000)))
        .scoreMode("total");
        
    }
    
    /**
     * 测试索引查询
     */
    @Test
    public void testIndicesQueryBuilder () {
        QueryBuilder queryBuilder = QueryBuilders.indicesQuery(
                QueryBuilders.termQuery("user", "kimchy"), "index1", "index2")
                .noMatchQuery(QueryBuilders.termQuery("user", "kimchy"));
        
    }
    
    
    
    /**
     * 查询遍历抽取
     * @param queryBuilder
     */
    private void searchFunction(QueryBuilder queryBuilder) {
        SearchResponse response = client.prepareSearch("twitter")
                .setSearchType(SearchType.DFS_QUERY_THEN_FETCH)
                .setScroll(new TimeValue(60000))
                .setQuery(queryBuilder)
                .setSize(100).execute().actionGet();
        
        while(true) {
            response = client.prepareSearchScroll(response.getScrollId())
                .setScroll(new TimeValue(60000)).execute().actionGet();
            for (SearchHit hit : response.getHits()) {
                Iterator<Entry<String, Object>> iterator = hit.getSource().entrySet().iterator();
                while(iterator.hasNext()) {
                    Entry<String, Object> next = iterator.next();
                    System.out.println(next.getKey() + ": " + next.getValue());
                    if(response.getHits().hits().length == 0) {
                        break;
                    }
                }
            }
            break;
        }
//        testResponse(response);
    }
    
    /**
     * 对response结果的分析
     * @param response
     */
    public void testResponse(SearchResponse response) {
        // 命中的记录数
        long totalHits = response.getHits().totalHits();
        
        for (SearchHit searchHit : response.getHits()) {
            // 打分
            float score = searchHit.getScore();
            // 文章id
            int id = Integer.parseInt(searchHit.getSource().get("id").toString());
            // title
            String title = searchHit.getSource().get("title").toString();
            // 内容
            String content = searchHit.getSource().get("content").toString();
            // 文章更新时间
            long updatetime = Long.parseLong(searchHit.getSource().get("updatetime").toString());
        }
    }
    
    /**
     * 对结果设置高亮显示
     */
    public void testHighLighted() {
        /*  5.0 版本后的高亮设置
         * client.#().#().highlighter(hBuilder).execute().actionGet();
        HighlightBuilder hBuilder = new HighlightBuilder();
        hBuilder.preTags("<h2>");
        hBuilder.postTags("</h2>");
        hBuilder.field("user");        // 设置高亮显示的字段
        */
        // 加入查询中
        SearchResponse response = client.prepareSearch("blog")
            .setQuery(QueryBuilders.matchAllQuery())
            .addHighlightedField("user")        // 添加高亮的字段
            .setHighlighterPreTags("<h1>")
            .setHighlighterPostTags("</h1>")
            .execute().actionGet();
        
        // 遍历结果, 获取高亮片段
        SearchHits searchHits = response.getHits();
        for(SearchHit hit:searchHits){
            System.out.println("String方式打印文档搜索内容:");
            System.out.println(hit.getSourceAsString());
            System.out.println("Map方式打印高亮内容");
            System.out.println(hit.getHighlightFields());

            System.out.println("遍历高亮集合，打印高亮片段:");
            Text[] text = hit.getHighlightFields().get("title").getFragments();
            for (Text str : text) {
                System.out.println(str.string());
            }
        }
    }
}
```

```java
import java.net.InetSocketAddress;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.Map.Entry;

import org.elasticsearch.action.ListenableActionFuture;
import org.elasticsearch.action.get.GetRequestBuilder;
import org.elasticsearch.action.get.GetResponse;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.action.search.SearchType;
import org.elasticsearch.client.transport.TransportClient;
import org.elasticsearch.common.settings.Settings;
import org.elasticsearch.common.text.Text;
import org.elasticsearch.common.transport.InetSocketTransportAddress;
import org.elasticsearch.common.unit.TimeValue;
import org.elasticsearch.index.query.IndicesQueryBuilder;
import org.elasticsearch.index.query.NestedQueryBuilder;
import org.elasticsearch.index.query.QueryBuilder;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.index.query.QueryStringQueryBuilder;
import org.elasticsearch.index.query.RangeQueryBuilder;
import org.elasticsearch.index.query.SpanFirstQueryBuilder;
import org.elasticsearch.index.query.WildcardQueryBuilder;
import org.elasticsearch.search.SearchHit;
import org.elasticsearch.search.SearchHits;
import org.junit.Before;
import org.junit.Test;

/**
 * 查询条件构造方法
 * @author xiaozm
 *
 */
public class ESQueryCondition {


    /**
     * 使用QueryBuilder
     * termQuery("key", obj) 完全匹配
     * termsQuery("key", obj1, obj2..)   一次匹配多个值
     * matchQuery("key", Obj) 单个匹配, field不支持通配符, 前缀具高级特性
     * multiMatchQuery("text", "field1", "field2"..);  匹配多个字段, field有通配符忒行
     * matchAllQuery();         匹配所有文件
     */
    @Test
    public void testQueryBuilder() {
//        QueryBuilder queryBuilder = QueryBuilders.termQuery("user", "kimchy");
　　　　　　QueryBUilder queryBuilder = QueryBuilders.termQuery("user", "kimchy", "wenbronk", "vini");
        QueryBuilders.termsQuery("user", new ArrayList<String>().add("kimchy"));
//        QueryBuilder queryBuilder = QueryBuilders.matchQuery("user", "kimchy");
//        QueryBuilder queryBuilder = QueryBuilders.multiMatchQuery("kimchy", "user", "message", "gender");
        QueryBuilder queryBuilder = QueryBuilders.matchAllQuery();
        searchFunction(queryBuilder);
        
    }
    
    /**
     * 组合查询
     * must(QueryBuilders) :   AND
     * mustNot(QueryBuilders): NOT
     * should:                  : OR
     */
    @Test
    public void testQueryBuilder2() {
        QueryBuilder queryBuilder = QueryBuilders.boolQuery()
            .must(QueryBuilders.termQuery("user", "kimchy"))
            .mustNot(QueryBuilders.termQuery("message", "nihao"))
            .should(QueryBuilders.termQuery("gender", "male"));
        searchFunction(queryBuilder);
    }
    
    /**
     * 只查询一个id的
     * QueryBuilders.idsQuery(String...type).ids(Collection<String> ids)
     */
    @Test
    public void testIdsQuery() {
        QueryBuilder queryBuilder = QueryBuilders.idsQuery().ids("1");
        searchFunction(queryBuilder);
    }
    
    /**
     * 包裹查询, 高于设定分数, 不计算相关性
     */
    @Test
    public void testConstantScoreQuery() {
        QueryBuilder queryBuilder = QueryBuilders.constantScoreQuery(QueryBuilders.termQuery("name", "kimchy")).boost(2.0f);
        searchFunction(queryBuilder);
        // 过滤查询
//        QueryBuilders.constantScoreQuery(FilterBuilders.termQuery("name", "kimchy")).boost(2.0f);
        
    }
    
    /**
     * disMax查询
     * 对子查询的结果做union, score沿用子查询score的最大值, 
     * 广泛用于muti-field查询
     */
    @Test
    public void testDisMaxQuery() {
        QueryBuilder queryBuilder = QueryBuilders.disMaxQuery()
            .add(QueryBuilders.termQuery("user", "kimch"))  // 查询条件
            .add(QueryBuilders.termQuery("message", "hello"))
            .boost(1.3f)
            .tieBreaker(0.7f);
        searchFunction(queryBuilder);
    }
    
    /**
     * 模糊查询
     * 不能用通配符, 找到相似的
     */
    @Test
    public void testFuzzyQuery() {
        QueryBuilder queryBuilder = QueryBuilders.fuzzyQuery("user", "kimch");
        searchFunction(queryBuilder);
    }
    
    /**
     * 父或子的文档查询
     */
    @Test
    public void testChildQuery() {
        QueryBuilder queryBuilder = QueryBuilders.hasChildQuery("sonDoc", QueryBuilders.termQuery("name", "vini"));
        searchFunction(queryBuilder);
    }
    
    /**
     * moreLikeThisQuery: 实现基于内容推荐, 支持实现一句话相似文章查询
     * {   
        "more_like_this" : {   
        "fields" : ["title", "content"],   // 要匹配的字段, 不填默认_all
        "like_text" : "text like this one",   // 匹配的文本
        }   
    }     
    
    percent_terms_to_match：匹配项（term）的百分比，默认是0.3

    min_term_freq：一篇文档中一个词语至少出现次数，小于这个值的词将被忽略，默认是2
    
    max_query_terms：一条查询语句中允许最多查询词语的个数，默认是25
    
    stop_words：设置停止词，匹配时会忽略停止词
    
    min_doc_freq：一个词语最少在多少篇文档中出现，小于这个值的词会将被忽略，默认是无限制
    
    max_doc_freq：一个词语最多在多少篇文档中出现，大于这个值的词会将被忽略，默认是无限制
    
    min_word_len：最小的词语长度，默认是0
    
    max_word_len：最多的词语长度，默认无限制
    
    boost_terms：设置词语权重，默认是1
    
    boost：设置查询权重，默认是1
    
    analyzer：设置使用的分词器，默认是使用该字段指定的分词器
     */
    @Test
    public void testMoreLikeThisQuery() {
        QueryBuilder queryBuilder = QueryBuilders.moreLikeThisQuery("user")
                            .like("kimchy");
//                            .minTermFreq(1)         //最少出现的次数
//                            .maxQueryTerms(12);        // 最多允许查询的词语
        searchFunction(queryBuilder);
    }
    
    /**
     * 前缀查询
     */
    @Test
    public void testPrefixQuery() {
        QueryBuilder queryBuilder = QueryBuilders.matchQuery("user", "kimchy");
        searchFunction(queryBuilder);
    }
    
    /**
     * 查询解析查询字符串
     */
    @Test
    public void testQueryString() {
        QueryBuilder queryBuilder = QueryBuilders.queryStringQuery("+kimchy");
        searchFunction(queryBuilder);
    }
    
    /**
     * 范围内查询
     */
    public void testRangeQuery() {
        QueryBuilder queryBuilder = QueryBuilders.rangeQuery("user")
            .from("kimchy")
            .to("wenbronk")
            .includeLower(true)     // 包含上界
            .includeUpper(true);      // 包含下届
        searchFunction(queryBuilder);
    }
    
    /**
     * 跨度查询
     */
    @Test
    public void testSpanQueries() {
         QueryBuilder queryBuilder1 = QueryBuilders.spanFirstQuery(QueryBuilders.spanTermQuery("name", "葫芦580娃"), 30000);     // Max查询范围的结束位置  
      
         QueryBuilder queryBuilder2 = QueryBuilders.spanNearQuery()  
                .clause(QueryBuilders.spanTermQuery("name", "葫芦580娃")) // Span Term Queries  
                .clause(QueryBuilders.spanTermQuery("name", "葫芦3812娃"))  
                .clause(QueryBuilders.spanTermQuery("name", "葫芦7139娃"))  
                .slop(30000)                                               // Slop factor  
                .inOrder(false)  
                .collectPayloads(false);  
  
        // Span Not
         QueryBuilder queryBuilder3 = QueryBuilders.spanNotQuery()  
                .include(QueryBuilders.spanTermQuery("name", "葫芦580娃"))  
                .exclude(QueryBuilders.spanTermQuery("home", "山西省太原市2552街道"));  
  
        // Span Or   
         QueryBuilder queryBuilder4 = QueryBuilders.spanOrQuery()  
                .clause(QueryBuilders.spanTermQuery("name", "葫芦580娃"))  
                .clause(QueryBuilders.spanTermQuery("name", "葫芦3812娃"))  
                .clause(QueryBuilders.spanTermQuery("name", "葫芦7139娃"));  
  
        // Span Term  
         QueryBuilder queryBuilder5 = QueryBuilders.spanTermQuery("name", "葫芦580娃");  
    }
    
    /**
     * 测试子查询
     */
    @Test
    public void testTopChildrenQuery() {
        QueryBuilders.hasChildQuery("tweet", 
                QueryBuilders.termQuery("user", "kimchy"))
            .scoreMode("max");
    }
    
    /**
     * 通配符查询, 支持 * 
     * 匹配任何字符序列, 包括空
     * 避免* 开始, 会检索大量内容造成效率缓慢
     */
    @Test
    public void testWildCardQuery() {
        QueryBuilder queryBuilder = QueryBuilders.wildcardQuery("user", "ki*hy");
        searchFunction(queryBuilder);
    }
    
    /**
     * 嵌套查询, 内嵌文档查询
     */
    @Test
    public void testNestedQuery() {
        QueryBuilder queryBuilder = QueryBuilders.nestedQuery("location", 
                QueryBuilders.boolQuery()
                    .must(QueryBuilders.matchQuery("location.lat", 0.962590433140581))
                    .must(QueryBuilders.rangeQuery("location.lon").lt(36.0000).gt(0.000)))
        .scoreMode("total");
        
    }
    
    /**
     * 测试索引查询
     */
    @Test
    public void testIndicesQueryBuilder () {
        QueryBuilder queryBuilder = QueryBuilders.indicesQuery(
                QueryBuilders.termQuery("user", "kimchy"), "index1", "index2")
                .noMatchQuery(QueryBuilders.termQuery("user", "kimchy"));
        
    }
    
    
    
    /**
     * 查询遍历抽取
     * @param queryBuilder
     */
    private void searchFunction(QueryBuilder queryBuilder) {
        SearchResponse response = client.prepareSearch("twitter")
                .setSearchType(SearchType.DFS_QUERY_THEN_FETCH)
                .setScroll(new TimeValue(60000))
                .setQuery(queryBuilder)
                .setSize(100).execute().actionGet();
        
        while(true) {
            response = client.prepareSearchScroll(response.getScrollId())
                .setScroll(new TimeValue(60000)).execute().actionGet();
            for (SearchHit hit : response.getHits()) {
                Iterator<Entry<String, Object>> iterator = hit.getSource().entrySet().iterator();
                while(iterator.hasNext()) {
                    Entry<String, Object> next = iterator.next();
                    System.out.println(next.getKey() + ": " + next.getValue());
                    if(response.getHits().hits().length == 0) {
                        break;
                    }
                }
            }
            break;
        }
//        testResponse(response);
    }
    
    /**
     * 对response结果的分析
     * @param response
     */
    public void testResponse(SearchResponse response) {
        // 命中的记录数
        long totalHits = response.getHits().totalHits();
        
        for (SearchHit searchHit : response.getHits()) {
            // 打分
            float score = searchHit.getScore();
            // 文章id
            int id = Integer.parseInt(searchHit.getSource().get("id").toString());
            // title
            String title = searchHit.getSource().get("title").toString();
            // 内容
            String content = searchHit.getSource().get("content").toString();
            // 文章更新时间
            long updatetime = Long.parseLong(searchHit.getSource().get("updatetime").toString());
        }
    }
    
    /**
     * 对结果设置高亮显示
     */
    public void testHighLighted() {
        /*  5.0 版本后的高亮设置
         * client.#().#().highlighter(hBuilder).execute().actionGet();
        HighlightBuilder hBuilder = new HighlightBuilder();
        hBuilder.preTags("<h2>");
        hBuilder.postTags("</h2>");
        hBuilder.field("user");        // 设置高亮显示的字段
        */
        // 加入查询中
        SearchResponse response = client.prepareSearch("blog")
            .setQuery(QueryBuilders.matchAllQuery())
            .addHighlightedField("user")        // 添加高亮的字段
            .setHighlighterPreTags("<h1>")
            .setHighlighterPostTags("</h1>")
            .execute().actionGet();
        
        // 遍历结果, 获取高亮片段
        SearchHits searchHits = response.getHits();
        for(SearchHit hit:searchHits){
            System.out.println("String方式打印文档搜索内容:");
            System.out.println(hit.getSourceAsString());
            System.out.println("Map方式打印高亮内容");
            System.out.println(hit.getHighlightFields());

            System.out.println("遍历高亮集合，打印高亮片段:");
            Text[] text = hit.getHighlightFields().get("title").getFragments();
            for (Text str : text) {
                System.out.println(str.string());
            }
        }
    }
}
```



### 二、常用命令

```bash
查看所有索引状态
curl '192.168.46.62:9200/_cat/indices?v'
查看_Mapping
curl -XGET 192.168.46.62:9200/km_youshu_app_order/_mapping/
删除index
curl -XDELETE http://192.168.46.62:9200/km_youshu_app_order
查询索引部分的数据
curl -XGET http://192.168.46.62:9200/km_youshu_app_order/_search
查询某index下某属性的值（可查询所有数据），并导出
curl -H "Content-Type: application/json" -XPOST '192.168.46.62:9200/km_youshu_app_coupon/_search?pretty' -d '{"query": { "match": { "coupon_name": "验证4" } },"size": 100}' -o 导出文件名称.txt
查看所有mapping模板
curl -XGET '192.168.46.62:9200/_cat/templates'
查看具体某模板格式
curl -XGET 192.168.46.62:9200/_template/ys_coupon_template?pretty
删除模板
curl -XDELETE '192.168.46.62:9200/_template/km_youshu_app_coupon_template?pretty'
动态新增映射字段
curl -X PUT "192.168.46.62:9200/km_youshu_app_order/_mapping?pretty" -H 'Content-Type: application/json' -d'
{
"properties": {
"big_category_id":{"type":"long"},
"big_category_name":{"type":"keyword"}
}
```

### 三、优化配置

```
1.设置分片副本数为0
index.number_of_replicas:0
```

#### 3.1 Elasticsearch 优化—写入优化

---



- Elasticsearch优化——写入优化
- - 1. translog flush间隔调整
  - 2. 索引刷新间隔refresh_interval
  - 3. 段合并优化
  - 4. indexing buffer
  - 5. 使用bulk请求
  - - 5.1 bulk线程池和队列
    - 5.2 并发执行bulk请求
  - 6. 磁盘间的任务均衡
  - 7. 节点间的任务均衡
  - 8. 索引过程调整和优化
  - - 8.1 自动生成doc ID
    - 8.2 调整字段mappings
    - 8.3 调整_source字段
    - 8.4 禁用_all字段
    - 8.5 对Analyzed的字段禁用Noms
    - 8.6 index_options设置
  - 9. 参考配置

---




在ES的默认设置下，是综合考虑数据可靠性、搜索实时性、写入速度等因素的。当离开默认设置、追求极致的写入速度时，很多是以牺牲可靠性和实时搜索性为代价的。有时候，业务上对数据可靠性和搜索实时性要求并不高，反而对写入熟读要求很高，此时可以调整一些策略，最大化写入速度。



接下来的优化基于集群正常运行的前提下，如果集群首次批量导入数据，则可以将副本数设置为0，导入完毕再将副本数调整回去，这样副本需要复制，节省索引过程。

综合来说，提升写入速度从以下几方面入手：

- 加大translog flush间隔，目的是降低iops、writeblock。
- 加大index refresh间隔，除了降低I/O，更重要的是降低segment merge频率。
- 调整bulk请求。
- 优化磁盘间的任务均匀情况，将shard尽量均匀分布到物理主机的各个磁盘。
- 优化节点间的任务分布，将任务尽量均匀的分发到各个节点。
- 优化Lucene层建立索引的过程，目的是降低CPU占用率及I/O，例如，禁用 _all字段。

## 1. translog flush间隔调整

从ES 2.x开始，在默认设置下，translog的持久化策略为：每个请求都“flush”。对应配置项如下：

```yaml
index.translog.durability: request
```

这是影响ES写入速度的最大因素。但是只有这样，写操作才有可能是可靠的。如果系统可以接受一定概率的数据丢失(例如，数据写入主分片成功，尚未复制到副本分片，主机断电。由于数据既没有刷到Lucene，translog也没有刷盘，恢复是translog中没有这个数据，数据丢失)，则调整translog持久化策略为周期性和一定大小的时候"flush"，例如：

```yaml
#设置为async表示translog的刷盘策略按sync_interval配置指定的时间周期进行
index.translog.durability: async
#加大translog刷盘间隔时间。默认5s，不可低于100ms。
index.translog.sync_interval: 120s
#超过这个大小会导致refresh操作，产生新的Lucene分段。默认值512MB
index.translog.flush_threshold_size: 1024mb
```

## 2. 索引刷新间隔refresh_interval

默认情况下索引的refresh_interval为1秒，这意味着数据写入1秒后就可以被搜索到，每次索引的refresh会产生一个新的Lucene段，这会导致频繁的segment merge行为，如果不需要这么高的搜索实时性应该降低索引refresh周期，例如：

```yaml
index.refresh_interval: 120s
```

或者

```json
PUT /my_index
{
    "settings":{
        "refresh_interval": "120s"
    }
}
```

## 3. 段合并优化

**segment merge**操作对系统**I/O**和内存占用都比较高，从ES 2.0开始，merge行为不再由ES控制，而是Lucene控制。因此以下配置被删除：

```yaml
indices.store.throttle.type 
indices.store.throttle.max_bytes_per_sec 
index.store.throttle.type 
index.store.throttle.max_bytes_per_sec
```

改为以下调整开关：

```yaml
index.merge.scheduler.max_thread_count: 4
index.merge.policy.*
```

最大线程数`max_thread_count`的默认值如下：

```tex
max_thread_count = Math.max(1, Math.min(4, Runtime.getRuntime().avaliableProcessors() / 2))
```

以上是一个比较理想的值，如果只有一个硬盘并且并非SSD，则应该把它设置为1，因为在旋转存储介质上并发写，由于寻址的原因，只会降低写入速度。

merge策略index.merge.policy有三种：

- tiered(默认策略)；
- log_byte_size；
- log_doc。

每个策略的具体描述可以参考：

目前我们使用默认策略，但是对策略的参数进行了调整。

索引创建时合并策略就已经确定，不能更改，但是可以动态更新策略参数。如果堆栈经常有很多merge，则可以尝试调整以下策略配置：

- `index.merge.policy.segments_per_tier`

  该属性指定了每层分段的数量，取值越小则segment越少，因此需要merge的操作更多，可以考虑适当增加此值。默认为10，其应该大于等于`index.merge.poliycy.max_merge_at_once`。

- `index.merge.policy.max_merged_segment`

  指定了单个segment的最大容量，默认为5GB，可以考虑适当降低此值。

## 4. indexing buffer

`ndex buffer`在为doc建立索引时使用，当缓冲满时会刷入磁盘，生成一个新的segment，这是除refresh_interval刷新索引外，另一个生成新segment的机会。每个shard有自己的`ndexing buffer`，下面的这个buffer大小的配置需要除以这个节点上所有shard的数量：

`indices.memory.index_buffer_size`：默认为整个堆空间的10%。

`indices.memory.min_index_buffer_size`：默认为48MB。

`indices.memory.max_index_buffer_size`：默认无限制。

在执行大量的索引操作时，`indices.memory.index_buffer_size`的默认值可能不够，这和可用堆内存、单节点上的shard数量相关，可以考虑适当增大该值。

## 5. 使用bulk请求

批量写比一个索引请求只写单个文档的效率高得多，但是要注意bulk请求的整体字节数不要太大，太大的请求可能会给集群带来内存压力，因此每个请求最好避免超过几十兆字节，即使较大的请求看上去执行的更好。

### 5.1 bulk线程池和队列

建立索引的过程属于计算密集型任务，应该使用固定大小的线程池，来不及处理的任务放入队列。线程池最大线程数量应配置为CPU核心数+1，这也是bulk线程池的默认配置，可以避免过多的上下文切换。队列大小可以适当增加，但一定要严格控制大小，过大的队列导致较高的GC压力，并可能导致FGC频繁发生。

### 5.2 并发执行bulk请求

bulk写请求是个长任务，为了给系统增加足够的写入压力，写入过程应该多个客户端、多线程的并行执行，。如果要验证系统的极限写入能力，那么目标就是把CPU压满。磁盘util、内存等一般都不是瓶颈。如果CPU没有压满，则应该提高写入端的并发数量。但是要注意bulk线程池队列的reject情况，出现regect代表ES的bulk队列满了，客户端请求被拒绝，此时客户端收到429错误(`TOO_MANY_REQUESTS`)，客户端对此的处理策略应该是延时重试。不可忽略这个异常，否则写入系统的数据会少于预期。即使客户端正确处理了429错误，我们仍然应该尽量避免产生reject。因此，在评估极限的写入能力时，客户端的极限写入并发量应该控制在不产生reject前提下的最大值为宜。

## 6. 磁盘间的任务均衡

如果不熟方案是为path.data配置多个路径来使用多块磁盘，ES通过下面2种策略均衡的写入不同的磁盘：

- 简单轮询：在系统初始化阶段，简单轮询的效果是最均匀的。
- 基于可用空间的动态加权轮询：以可用空间作为权重，在磁盘之间加权轮询。

## 7. 节点间的任务均衡

为了节点间的任务尽量均衡，数据写入客户端应该把bulk请求轮询发送到各个节点，当使用REST API的bulk接口发送数据时，客户端将会轮询发送到集群节点，在创建客户端对象时添加节点。

## 8. 索引过程调整和优化

### 8.1 自动生成doc ID

通过ES写入流程可以看出，写入doc时如果外部指定了id，则es会尝试读取原来doc的版本号，以判断是否需要更新。这会涉及一次读取磁盘操作，通过自动生成doc ID可以避免这个环节。

### 8.2 调整字段mappings

1. 减少字段数量，对于不需要建立索引的字段不写入ES。
2. 将不需要建立索引的字段index属性设置为not_analyzed或no。对字段不分词，或者不索引，可以减少很多运算操作，降低CPU占用。尤其是binary类型，默认情况下占用CPU非常高，而这种类型进行分词通常没有意义。
3. 减少字段内容长度，如果原始数据的大段内容无需建立索引，则尽量减少不必要的内容。
4. 使用不同的分析器(analyzer)，不同的分析器在索引过程中运算复杂度也有较大的差异。

### 8.3 调整_source字段

_source字段用于存储doc原始数据，对于不需要存储的字段，可以通过includes excludes过滤，或者将 _source禁用，一般用于索引和数据分离。

这样可以降低**I/O**的压力，不过实际场景中大多不会禁用 _source，即使过滤掉某些字段，对于写入速度提升作用也不大，满负荷写入情况下，基本是CPU先跑满，瓶颈在于CPU。

### 8.4 禁用_all字段

从 ES 6.0开始， _all字段默认不启用，而在此前的版本中， _all字段默认是开启的。 _all字段中包含所有字段分词后的关键词，作用是可以在搜索的时候不指定特定字段，从所有字段中检索。ES 6.0默认禁用 _all的主要原因有以下几点：

- 由于需要从其他的全部字段复制所有字段值，导致 _all字段占用非常大的空间。
- _all字段有自己的分析器，在进行某些查询时(例如，同义词)，结果不符合预期，因为没有匹配同一个分析器。
- 由于数据重复引起的额外建立索引的开销。
- 想要调试时，其内容不容易检查。
- 有些用户甚至不知道存在这个字段，导致了查询混乱。
- 有更改的替代方案。

在ES 6.0之前的版本中，可以在mapping中将enabled设置为false来禁用 _all字段：

```json
PUT /my_index
{
    "mappings":{
        "my_type":{
            "_all":{
                "enabled":false
            }
        }
    }
}
```

禁用 _all字段可以明显降低对CPU和**I/O**的压力。

### 8.5 对Analyzed的字段禁用Noms

Norms用于在搜索时计算doc的评分，如果不需要评分，则可以将其禁用：

```json
PUT my_index/_mapping/my_type
{
    "properties":{
        "title":{
            "type":"keyword",
            "norms":{
                "enabled":false
            }
        }
    }
}
```

### 8.6 index_options设置

`index_options`用于控制在建立倒排索引的过程中，哪些内容会被添加到倒排索引，例如，doc数量、词频、options、offset等信息，优化这些设置可以一定程度降低索引过程中的运算任务，节省CPU占用率。

不过在实际场景中，通常很难确定业务将来会不会用到这些信息，除非一开始方案就明确是这样设计的。

## 9. 参考配置

索引级的设想需要写在模板中，或者在创建索引是指定。

```json
{
    "template":"*",
    "order":0,
    "settings":{
        //单个分段的最大容量
        "index.merge.policy.max_merged_segment":"2gb",
        //每层分段的数量，值越小，合并操作越多，可以适当增加，默认10
        //不要小于max_merge_at_once
        "index.merge.policy.segments_per_tier":"24",
        //索引刷入操作系统缓存时间，默认1秒
        "index.refresh_interval":"120s",
        //索引刷入操作系统缓存策略，默认request，改为定时刷新
        "index.translog.durability":"async",
        //索引刷入磁盘的translog的阈值，默认512mb，提交commit point
        "index.translog.flush_threshold_size":"512mb",
        //日志刷新磁盘的时间，默认5s
        "index.translog.sync_interval":"120s",
        //索引分配延迟时间，默认1分钟
        "index.unassigned.node_left.delayed_timeout":"5d"
    }
}
```

elasticsearch.yml中的配置：

```yaml
#索引buffer大小，默认可使用堆的10%
indices.memory.index_buffer_size: 30%
```

