# Paimon Spark Catalog 未注册报错与命名机制

## 背景

在 Spark ThriftServer（EMR Serverless Spark，Spark 3.5.2 / Scala 2.12）上用三段式表名查 Paimon 表：

```sql
select * from <my_catalog>.<db>.<table>
```

报错：

```
java.lang.IllegalArgumentException: Current catalog is spark_catalog,
catalog <my_catalog> does not exist or Paimon only support single namespace,
but got [<my_catalog>, <db>]
```

同一会话里两段式查 Hive 表（走 `spark_catalog`）正常，只有带 catalog 前缀的查询失败。

## 核心问题：Spark 多段标识符的解析规则

Spark（DataSourceV2）对 `a.b.c` 这种多段标识符的解析顺序：

1. 先看第一段 `a` 是否是**已注册的 catalog**（即是否存在 `spark.sql.catalog.a` 配置）；
2. **匹配不到 catalog 时，不会报"catalog 不存在"**，而是回退到当前 catalog，把 `[a, b]` 整体当作 namespace、`c` 当作表名传进去。

于是 Paimon 的 SparkCatalog 收到一个两层 namespace，在 `CatalogUtils.checkNamespace` 处触发 `namespace.length == 1` 校验失败：

```java
// paimon-spark-common .../spark/utils/CatalogUtils.java
public static void checkNamespace(String[] namespace, String catalogName) {
    checkArgument(
            namespace.length == 1,
            "Current catalog is %s, catalog %s does not exist or Paimon only support single namespace, but got %s",
            ...);
}
```

**关键判断**：报错文案里 "catalog X does not exist" 是社区专门为这种场景加的提示——看到 "only support single namespace" 且 namespace 恰好是 `[catalog名, 库名]` 两段，第一优先怀疑不是 SQL 写错，而是 **catalog 没在这个会话里注册**。旁证方式：对比同会话里两段式查询是否正常。

## 修复：注册 catalog

在任务/会话的 Spark 配置里补上（filesystem catalog on OSS 示例）：

```
spark.sql.catalog.<my_catalog>                      org.apache.paimon.spark.SparkCatalog
spark.sql.catalog.<my_catalog>.warehouse            oss://<bucket>/<warehouse_path>
spark.sql.catalog.<my_catalog>.fs.oss.accessKeyId   <YOUR_AK>
spark.sql.catalog.<my_catalog>.fs.oss.accessKeySecret <YOUR_SK>
spark.sql.catalog.<my_catalog>.fs.oss.endpoint      oss-<region>.aliyuncs.com
```

若元数据在 Hive Metastore，则改配 `metastore=hive` + `uri=thrift://<hms-host>:9083`。

补充：Spark 的 catalog 是**惰性初始化**的，理论上在已有会话里 `SET spark.sql.catalog.xxx=...` 也能生效，前提是 paimon-spark jar 已在 classpath 上（报错栈能走到 Paimon 代码即说明 jar 在）。

## catalog 名字可以随便起吗？可以

`spark.sql.catalog.<名字>` 是 **Spark 的 catalog plugin 机制**，名字完全由用户定义，Paimon 只是被动接收：

```java
// SparkCatalog.java
public void initialize(String name, CaseInsensitiveStringMap options) {
    this.catalogName = name;   // Spark 把配置 key 里的名字原样传入
```

- Spark 实例化 catalog 时，把 `spark.sql.catalog.` 后面那段作为 `name` 传给 `initialize()`；子配置 `spark.sql.catalog.<名字>.*` 收集成 `options` 一并传入。Paimon 源码里没有任何地方校验名字必须是 `paimon`（官方文档只是示例命名）。
- **唯一有特殊含义的名字是 `spark_catalog`**（Spark 内置 session catalog）。想让 Paimon 接管它、实现不带前缀查表，要用 `org.apache.paimon.spark.SparkGenericCatalog` 而非 `SparkCatalog`。
- 可以同时注册多个不同名字的 Paimon catalog 指向不同 warehouse，互不冲突。

## 关键知识点

- 排查这类报错的高效路径：sparse clone 源码 + grep 报错文案，直接定位抛错点，比猜配置快得多：
  ```bash
  git clone --depth 1 --filter=blob:none --sparse https://github.com/apache/paimon.git
  git sparse-checkout set paimon-spark
  grep -rn "only support single namespace" paimon-spark/
  ```
- Spark 三段式标识符解析失败时的"错位报错"模式：第一段不是注册的 catalog 时，错误会以"namespace 非法"的形式出现在**当前 catalog** 的实现里，而不是直接说 catalog 找不到。这个模式对其他 DSv2 数据源（Iceberg 等）同样适用。
