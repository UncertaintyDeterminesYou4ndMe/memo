# 仓库概述

本仓库主要记录了关于 Kafka、Hbase、Kubernetes 以及数仓等相关技术的命令、配置和一些运维场景说明，方便开发人员和运维人员参考和使用。

## 目录结构

目前仓库中的主要内容包含在 `memo/memo.md` 文件中，该文件按不同的主题和技术进行了详细的记录，主要涵盖以下几个方面：

### Kafka 相关
- **运维操作**：包括增加 topic 数、负载不均衡的 topic 手动迁移等场景及对应的命令。
- **常用命令**：如查看分区消费堆积量、查看 Broker 磁盘信息等常用 Kafka 命令。
- **配置信息**：Kafka 的一些配置参数，如 `log.dirs`、`log.retention.hours` 等。

### Hbase 相关
对 Hbase 有简单的提及，包括其在聚合操作时需要使用 Phoenix 等说明。

### Kubernetes 相关
记录了 Kubernetes 中一些资源的配置信息，如 ConfigMap、Pod 状态等。

### 数仓相关
- **数仓分层**：介绍了数仓的分层结构，包括 ODS、DWD、DWS、DWT、ADS 各层的含义。
- **技术选型**：说明了 mysql、clickhouse、es、hbase 在数仓中的主要用途，如 mysql 用于 OLTP，clickhouse 用于 OLAP 等。
