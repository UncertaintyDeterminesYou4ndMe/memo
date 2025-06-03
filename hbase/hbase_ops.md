## 均衡 region

## 修复 region 异常关闭（abnormal closed）
hbck
closed
unassign

## hbase 查看 cpu 消耗线程

## hbase 查看 内存使用

## hbase 均衡开关
若出现 hbase 重启失败，务必手动开启 `hbase balancer_enabled` 开关
```
hbase shell > `balancer_enabled`
hbase shell > `balance_switch true`
```
