
- Topic 管理（创建、删除、更新、列出、详情、消费、闲置检测、分析）
- 消费者组管理（列出、详情、删除）
- 只依赖系统 JDK 和本地 Kafka 包（无需 jdk-11.0.29）

请将此脚本保存为 `kafka_cluster_manager.sh`，放在 `kafka_manager` 目录下，与 `kafka_2.13-3.8.1` 同级。

---

```bash
#!/bin/bash
# Kafka Topic & Consumer Group 管理脚本（增强版，JDK自适应）

# 颜色
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

echo_ok()   { echo -e "${GREEN}[OK]${NC} $1"; }
echo_err()  { echo -e "${RED}[ERROR]${NC} $1" >&2; }
echo_warn() { echo -e "${YELLOW}[WARNING]${NC} $1"; }
echo_info() { echo -e "${BLUE}[INFO]${NC} $1"; }

# Kafka目录
KAFKA_DIR="kafka_2.13-3.8.1"

# 集群配置
declare -A CLUSTERS=(
    ["quote"]="xxxx:9092"
    ["bi"]="xxx:9092"
    ["core"]="xxx:9092"
)

# 检查依赖
check_dependencies() {
    echo_info "检查依赖..."
    if [ ! -d "$KAFKA_DIR" ]; then
        echo_err "Kafka目录不存在: $KAFKA_DIR"
        return 1
    fi
    if [ ! -f "$KAFKA_DIR/bin/kafka-topics.sh" ]; then
        echo_err "Kafka脚本不存在: $KAFKA_DIR/bin/kafka-topics.sh"
        return 1
    fi
    echo_ok "依赖检查通过"
    return 0
}

# 设置环境
setup_env() {
    export KAFKA_HOME="$(pwd)/$KAFKA_DIR"
    export PATH="$KAFKA_HOME/bin:$PATH"
    echo_ok "Kafka环境设置成功"
}

# 获取 Bootstrap Servers
get_bootstrap_servers() {
    local cluster=$1
    echo "${CLUSTERS[$cluster]}"
}

# 列出所有 Topic
list_topics() {
    local cluster=$1
    local bootstrap
    bootstrap=$(get_bootstrap_servers "$cluster")
    echo_info "列出所有Topic（集群: $cluster）"
    $KAFKA_HOME/bin/kafka-topics.sh --list --bootstrap-server "$bootstrap"
}

# 查看 Topic 详情
describe_topic() {
    local cluster=$1
    local topic=$2
    local bootstrap
    bootstrap=$(get_bootstrap_servers "$cluster")
    echo_info "Topic详情（集群: $cluster, Topic: $topic）"
    $KAFKA_HOME/bin/kafka-topics.sh --describe --topic "$topic" --bootstrap-server "$bootstrap"
}

# 创建 Topic
create_topic() {
    local cluster=$1
    local topic=$2
    local partitions=${3:-1}
    local replication=${4:-1}
    local retention_ms=${5:-}
    local bootstrap
    bootstrap=$(get_bootstrap_servers "$cluster")
    echo_info "创建Topic（集群: $cluster, Topic: $topic, 分区: $partitions, 副本: $replication）"
    local cmd=("$KAFKA_HOME/bin/kafka-topics.sh" --create --topic "$topic" --partitions "$partitions" --replication-factor "$replication" --bootstrap-server "$bootstrap")
    if [ -n "$retention_ms" ]; then
        cmd+=(--config "retention.ms=$retention_ms")
    fi
    "${cmd[@]}"
}

# 删除 Topic
delete_topic() {
    local cluster=$1
    local topic=$2
    local bootstrap
    bootstrap=$(get_bootstrap_servers "$cluster")
    echo_warn "删除Topic（集群: $cluster, Topic: $topic）"
    $KAFKA_HOME/bin/kafka-topics.sh --delete --topic "$topic" --bootstrap-server "$bootstrap"
}

# 更新 Topic 配置
update_topic_config() {
    local cluster=$1
    local topic=$2
    local configs=$3
    local bootstrap
    bootstrap=$(get_bootstrap_servers "$cluster")
    echo_info "更新Topic配置（集群: $cluster, Topic: $topic, 配置: $configs）"
    $KAFKA_HOME/bin/kafka-configs.sh --bootstrap-server "$bootstrap" --entity-type topics --entity-name "$topic" --alter --add-config "$configs"
}

# 消费 Topic 消息
consume_topic() {
    local cluster=$1
    local topic=$2
    local count=${3:-10}
    local bootstrap
    bootstrap=$(get_bootstrap_servers "$cluster")
    echo_info "消费Topic消息（集群: $cluster, Topic: $topic, 数量: $count）"
    $KAFKA_HOME/bin/kafka-console-consumer.sh --bootstrap-server "$bootstrap" --topic "$topic" --max-messages "$count" --property print.timestamp=true --property print.key=true
}

# 检查闲置 Topic（只简单检测无消费者组和无消息）
check_idle_topics() {
    local cluster=$1
    local bootstrap
    bootstrap=$(get_bootstrap_servers "$cluster")
    echo_info "检查闲置Topic（集群: $cluster）"
    local topics=$($KAFKA_HOME/bin/kafka-topics.sh --list --bootstrap-server "$bootstrap")
    for topic in $topics; do
        local msg_count=$($KAFKA_HOME/bin/kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list "$bootstrap" --topic "$topic" --time -1 2>/dev/null | awk -F: '{sum+=$3} END{print sum}')
        local consumer_groups=$($KAFKA_HOME/bin/kafka-consumer-groups.sh --bootstrap-server "$bootstrap" --list 2>/dev/null | grep -c .)
        if [ "$msg_count" -eq 0 ] || [ "$consumer_groups" -eq 0 ]; then
            echo_warn "闲置Topic: $topic (消息数: $msg_count, 消费者组: $consumer_groups)"
        fi
    done
}

# 分析 Topic 使用情况（统计分区、副本、内部Topic等）
analyze_topics() {
    local cluster=$1
    local bootstrap
    bootstrap=$(get_bootstrap_servers "$cluster")
    echo_info "分析Topic使用情况（集群: $cluster）"
    local topics=$($KAFKA_HOME/bin/kafka-topics.sh --list --bootstrap-server "$bootstrap")
    local total=0
    local internal=0
    for topic in $topics; do
        total=$((total+1))
        [[ "$topic" =~ ^__ ]] && internal=$((internal+1))
    done
    echo "总Topic数: $total"
    echo "内部Topic数: $internal"
    echo "用户Topic数: $((total-internal))"
}

# ================= 消费者组相关功能 =================

# 列出所有消费者组
list_consumer_groups() {
    local cluster=$1
    local bootstrap
    bootstrap=$(get_bootstrap_servers "$cluster")
    echo_info "列出所有消费者组（集群: $cluster）"
    $KAFKA_HOME/bin/kafka-consumer-groups.sh --bootstrap-server "$bootstrap" --list
}

# 查看消费者组详情
describe_consumer_group() {
    local cluster=$1
    local group=$2
    local bootstrap
    bootstrap=$(get_bootstrap_servers "$cluster")
    echo_info "消费者组详情（集群: $cluster, 组: $group）"
    $KAFKA_HOME/bin/kafka-consumer-groups.sh --bootstrap-server "$bootstrap" --group "$group" --describe
}

# 删除消费者组
delete_consumer_group() {
    local cluster=$1
    local group=$2
    local bootstrap
    bootstrap=$(get_bootstrap_servers "$cluster")
    echo_warn "删除消费者组（集群: $cluster, 组: $group）"
    $KAFKA_HOME/bin/kafka-consumer-groups.sh --bootstrap-server "$bootstrap" --delete --group "$group"
}

# ================= 主入口 =================

main() {
    if ! check_dependencies; then
        exit 1
    fi
    setup_env

    case "$1" in
        list)
            if [ "$2" != "-c" ] || [ -z "$3" ]; then
                echo_err "用法: $0 list -c <cluster>"
                exit 1
            fi
            list_topics "$3"
            ;;
        describe)
            if [ "$2" != "-c" ] || [ -z "$3" ] || [ "$4" != "-t" ] || [ -z "$5" ]; then
                echo_err "用法: $0 describe -c <cluster> -t <topic>"
                exit 1
            fi
            describe_topic "$3" "$5"
            ;;
        create)
            if [ "$2" != "-c" ] || [ -z "$3" ] || [ "$4" != "-t" ] || [ -z "$5" ]; then
                echo_err "用法: $0 create -c <cluster> -t <topic> [-p <partitions>] [-r <replication>] [--retention <ms>]"
                exit 1
            fi
            local partitions=1
            local replication=1
            local retention_ms=""
            for ((i=6;i<=$#;i++)); do
                arg="${!i}"
                case "$arg" in
                    -p) partitions="${!((i+1))}";;
                    -r) replication="${!((i+1))}";;
                    --retention) retention_ms="${!((i+1))}";;
                esac
            done
            create_topic "$3" "$5" "$partitions" "$replication" "$retention_ms"
            ;;
        delete)
            if [ "$2" != "-c" ] || [ -z "$3" ] || [ "$4" != "-t" ] || [ -z "$5" ]; then
                echo_err "用法: $0 delete -c <cluster> -t <topic>"
                exit 1
            fi
            delete_topic "$3" "$5"
            ;;
        update)
            if [ "$2" != "-c" ] || [ -z "$3" ] || [ "$4" != "-t" ] || [ -z "$5" ] || [ "$6" != "--config" ] || [ -z "$7" ]; then
                echo_err "用法: $0 update -c <cluster> -t <topic> --config <key=value,...>"
                exit 1
            fi
            update_topic_config "$3" "$5" "$7"
            ;;
        consume)
            if [ "$2" != "-c" ] || [ -z "$3" ] || [ "$4" != "-t" ] || [ -z "$5" ]; then
                echo_err "用法: $0 consume -c <cluster> -t <topic> [-n <count>]"
                exit 1
            fi
            local count=10
            if [ "$6" == "-n" ] && [ -n "$7" ]; then
                count="$7"
            fi
            consume_topic "$3" "$5" "$count"
            ;;
        check-idle)
            if [ "$2" != "-c" ] || [ -z "$3" ]; then
                echo_err "用法: $0 check-idle -c <cluster>"
                exit 1
            fi
            check_idle_topics "$3"
            ;;
        analyze)
            if [ "$2" != "-c" ] || [ -z "$3" ]; then
                echo_err "用法: $0 analyze -c <cluster>"
                exit 1
            fi
            analyze_topics "$3"
            ;;
        list-groups)
            if [ "$2" != "-c" ] || [ -z "$3" ]; then
                echo_err "用法: $0 list-groups -c <cluster>"
                exit 1
            fi
            list_consumer_groups "$3"
            ;;
        describe-group)
            if [ "$2" != "-c" ] || [ -z "$3" ] || [ "$4" != "-g" ] || [ -z "$5" ]; then
                echo_err "用法: $0 describe-group -c <cluster> -g <group>"
                exit 1
            fi
            describe_consumer_group "$3" "$5"
            ;;
        delete-group)
            if [ "$2" != "-c" ] || [ -z "$3" ] || [ "$4" != "-g" ] || [ -z "$5" ]; then
                echo_err "用法: $0 delete-group -c <cluster> -g <group>"
                exit 1
            fi
            delete_consumer_group "$3" "$5"
            ;;
        *)
            echo "用法:"
            echo "  $0 list -c <cluster>"
            echo "  $0 describe -c <cluster> -t <topic>"
            echo "  $0 create -c <cluster> -t <topic> [-p <partitions>] [-r <replication>] [--retention <ms>]"
            echo "  $0 delete -c <cluster> -t <topic>"
            echo "  $0 update -c <cluster> -t <topic> --config <key=value,...>"
            echo "  $0 consume -c <cluster> -t <topic> [-n <count>]"
            echo "  $0 check-idle -c <cluster>"
            echo "  $0 analyze -c <cluster>"
            echo "  $0 list-groups -c <cluster>"
            echo "  $0 describe-group -c <cluster> -g <group>"
            echo "  $0 delete-group -c <cluster> -g <group>"
            echo "支持的集群: quote, bi, core"
            ;;
    esac
}

main "$@"
```

---

## 使用示例

- **列出所有 Topic**
  ```bash
  ./kafka_cluster_manager.sh list -c bi
  ```

- **查看 Topic 详情**
  ```bash
  ./kafka_cluster_manager.sh describe -c bi -t your_topic_name
  ```

- **创建 Topic**
  ```bash
  ./kafka_cluster_manager.sh create -c bi -t mytopic -p 3 -r 2 --retention 21600000
  ```

- **删除 Topic**
  ```bash
  ./kafka_cluster_manager.sh delete -c bi -t mytopic
  ```

- **更新 Topic 配置**
  ```bash
  ./kafka_cluster_manager.sh update -c bi -t mytopic --config cleanup.policy=compact,retention.ms=21600000
  ```

- **消费 Topic 消息**
  ```bash
  ./kafka_cluster_manager.sh consume -c bi -t mytopic -n 20
  ```

- **检查闲置 Topic**
  ```bash
  ./kafka_cluster_manager.sh check-idle -c bi
  ```

- **分析 Topic 使用情况**
  ```bash
  ./kafka_cluster_manager.sh analyze -c bi
  ```

- **列出所有消费者组**
  ```bash
  ./kafka_cluster_manager.sh list-groups -c bi
  ```

- **查看消费者组详情**
  ```bash
  ./kafka_cluster_manager.sh describe-group -c bi -g your_group_name
  ```

- **删除消费者组**
  ```bash
  ./kafka_cluster_manager.sh delete-group -c bi -g your_group_name
  ```
  
