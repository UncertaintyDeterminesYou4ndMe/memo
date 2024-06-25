# 一. CDC
## 流行的 CDC 工具和技术
### 以下是一些广泛使用的 CDC 工具，用于实现实时数据集成和同步：

Debezium[1]：开源 CDC 平台，基于Apache Kafka 构建。它从 MySQL、PostgreSQL、Oracle、SQL Server 和 MongoDB 等数据库捕获并流式传输数据变化，提供可扩展且可靠的解决方案。

Maxwell CDC[2]：CDC 工具，将 MySQL 数据库的实时变化流式传输到Kafka、Kinesis 等平台，以其简单、易于设置和轻量架构著称，能高效地流式传输行级变化为 JSON。

Canal[3]：CDC 工具，能将 MySQL 数据库的实时变化流式传输到其他系统，提供统一的变更日志格式方案，并支持 JSON 和 protobuf 消息序列化。

Oracle GoldenGate[4]：Oracle 的 CDC 工具，用于跨 Oracle 数据库及 SQL Server、DB2 和 Teradata 等其他数据库实现实时数据复制和集成，提供高性能和低影响的数据捕获。

AWS数据库迁移服务（DMS）[5]：支持完全托管的 AWS 服务，支持各种数据库的迁移和复制，具备 CDC 功能，支持 Amazon RDS、Aurora 和本地数据库。

Microsoft SQL Server Change Data Capture[6]：SQL Server 内置的 CDC 功能，允许捕获和跟踪 SQL Server 数据库中表的变化。
