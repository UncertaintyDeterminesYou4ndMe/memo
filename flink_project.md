# 找不到类
```
find ~/.m2/repository -name "*flink-cdc*" -type d
```
```
find ~/.m2/repository/org/apache/flink/flink-cdc-common -name "*.jar"
```
```
jar -tf ~/.m2/repository/org/apache/flink/flink-cdc-common/3.2.1/flink-cdc-common-3.2.1.jar | grep UserDefinedFunction
```
