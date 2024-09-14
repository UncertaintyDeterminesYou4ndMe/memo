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
package com.shulidata.marketing.common.integration.elasticsearch.config;

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
package com.shulidata.marketing.common.integration.elasticsearch.util;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.shulidata.common.base.exception.CommonException;
import com.shulidata.common.utils.LoggerUtil;
import com.shulidata.marketing.common.model.base.ErrorCode;
import com.shulidata.marketing.common.model.elasticsearch.base.ESBaseData;
import com.shulidata.marketing.common.model.elasticsearch.base.ESPage;
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
 *
 * @author kaifang.z on 2023/4/10
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

