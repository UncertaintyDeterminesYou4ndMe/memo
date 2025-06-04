# kubectl 命令速查手册

## 基础命令

### 查看资源
```bash
kubectl get <resource> [flags]          # 列出资源（如 pods/deployments/services）
kubectl get pods -n <namespace>        # 查看指定命名空间的 Pod
kubectl get pods --all-namespaces      # 查看所有命名空间的 Pod
kubectl get pods -o wide               # 显示详细信息（IP/节点等）
kubectl get pods -o yaml/json          # 以 YAML/JSON 格式输出
kubectl describe pod <pod-name>        # 查看 Pod 详细信息（事件/状态等）

### 创建/删除资源
kubectl create -f <file.yaml>          # 通过 YAML 文件创建资源
kubectl delete pod <pod-name>          # 删除 Pod
kubectl delete -f <file.yaml>          # 删除 YAML 定义的资源
kubectl apply -f <file.yaml>           # 声明式更新资源

## 调试命令
### 日志查看
kubectl logs <pod-name>                # 查看 Pod 日志
kubectl logs -f <pod-name>             # 实时日志流（类似 tail -f）
kubectl logs --previous <pod-name>     # 查看崩溃容器的日志
kubectl logs -c <container-name> <pod-name>  # 多容器 Pod 指定容器日志
### 进入容器
kubectl exec -it <pod-name> -- /bin/sh   # 进入 Pod 的 Shell
kubectl exec <pod-name> -- <command>     # 在 Pod 中执行单次命令

### 端口转发
kubectl port-forward <pod-name> 8080:80  # 将本地 8080 转发到 Pod 的 80 端口

## 高级操作
### 资源管理
kubectl scale deploy <deploy-name> --replicas=3  # 扩容 Deployment
kubectl rollout status deploy/<deploy-name>      # 查看滚动更新状态
kubectl rollout undo deploy/<deploy-name>        # 回滚 Deployment
kubectl cordon <node-name>                       # 标记节点不可调度
kubectl drain <node-name>                        # 清空节点（维护用）

### 配置管理
kubectl config view                     # 查看当前 kubeconfig
kubectl config use-context <context>    # 切换集群上下文
kubectl get nodes --show-labels         # 查看节点标签
kubectl label nodes <node-name> key=value  # 给节点打标签

## 实用技巧
### 字段过滤
kubectl get pods --field-selector status.phase=Running  # 按状态过滤
kubectl get pods -l app=nginx          # 按标签筛选
### 资源监控
kubectl top pods                       # 查看 Pod 资源使用（需 metrics-server）
kubectl top nodes                      # 查看节点资源使用
### 快速生成模板
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml  # 生成 Pod YAML
kubectl create deploy nginx --image=nginx --dry-run=client -o yaml   # 生成 Deployment YAML


### 使用提示
1. 大部分命令支持 `-n <namespace>` 指定命名空间
2. 使用 `--help` 查看子命令帮助（如 `kubectl get --help`）
3. 组合命令时可通过 `| grep` 或 `-o json | jq` 进一步处理输出

> 注：需要安装 [jq](https://stedolan.github.io/jq/) 工具处理 JSON 输出

