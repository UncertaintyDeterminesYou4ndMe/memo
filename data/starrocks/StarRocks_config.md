# StarRocks 配置参数说明

## fe 动态参数
|参数|默认值|描述|
|---|---|---|
|qe_slow_log_ms|5000|Slow query的认定时长，单位为ms。|
|catalog_try_lock_timeout_ms|5000|Catalog Lock获取的超时时长，单位为ms。|
|edit_log_roll_num|50000|Image日志拆分大小。|
|ignore_unknown_log_id|false|当FE回滚到低版本时，可能存在低版本BE无法识别的logID。取值为true时，FE会忽略这些未知的logID；取值为false时，针对未知的logID，FE会退出进程。|
|ignore_meta_check|false|是否忽略元数据落后的情形。取值为false时忽略；取值为true时不忽略。|
|max_backend_down_time_second|3600|BE和FE失联之后，FE能够容忍BE重新加回来的最长时间，单位为s。|
|drop_backend_after_decommission|true|BE被下线后，是否删除该BE。取值为true时，BE被下线后立即删除该BE；取值为false时，BE被下线后不删除该BE。|
|expr_children_limit|10000|查询中IN谓词中可以涉及的数目。|
|expr_depth_limit|3000|查询嵌套的层次。|
|max_allowed_in_element_num_of_delete|10000|DELETE语句中IN谓词最多允许的元素数量。|
|max_layout_length_per_row|2147483647|单行最大的长度。|
|disable_cluster_feature|TRUE|是否禁用逻辑集群功能。取值为TRUE时禁用；取值为FALSE时不禁用。|
|enable_materialized_view|TRUE|是否允许创建物化视图。取值为TRUE时允许；取值为FALSE时不允许。|
|enable_decimal_v3|TRUE|是否开启Decimal V3。取值为TRUE时开启；取值为FALSE时不开启。|
|enable_sql_blacklist|FALSE|是否开启SQL Query黑名单校验。取值为TRUE时开启，黑名单中的Query不能被执行；取值为FALSE时不开启。|
|dynamic_partition_check_interval_seconds|600|动态分区检查的时间周期，单位为s。|
|dynamic_partition_enable|TRUE|是否开启动态分区功能。取值为TRUE时开启动态分区功能，可按需为新数据动态创建分区，同时StarRocks会自动删除过期分区；取值为FALSE时不开启动态分区功能。|
|max_partitions_in_one_batch|4096|批量创建分区时，分区数目的最大值。|
|max_query_retry_time|2|FE上查询重试的次数。|
|max_create_table_timeout_second|600|建表最大超时时间，单位为s。|
|max_running_rollup_job_num_per_table|1|每个Table执行Rollup任务的最大并发度。|
|max_planner_scalar_rewrite_num|100000|优化器重写ScalarOperator允许的最大次数。|
|statistics_manager_sleep_time_sec|60|统计信息相关元数据调度间隔周期，单位为s。|
|statistic_collect_interval_sec|300|自动定期采集任务中，检测数据更新的间隔时间，单位为s。|
|statistic_update_interval_sec|86400|统计信息Job的默认收集间隔时间，单位为s。|
|statistic_sample_collect_rows|200000|采样统计信息Job的默认采样行数，默认为200000行。|
|enable_statistic_collect|TRUE|统计信息收集功能开关。取值为TRUE时开启统计信息收集功能；取值为FALSE时不开启。|
|enable_local_replica_selection|FALSE|优化器是否优先选择与该FE相同IP的BE节点上的tablet。取值为TRUE时是；取值为FALSE时否。|
|max_distribution_pruner_recursion_depth|100|分区裁剪允许的最大递归深度。|
|load_straggler_wait_second|300|控制BE副本最大容忍的导入落后时长，单位为s。超过该时长则进行克隆。|
|desired_max_waiting_jobs|100|最多等待的任务，适用于所有的任务，如建表、导入和Schema Change。|
|max_running_txn_num_per_db|100|并发导入的任务数。|
|max_load_timeout_second|259200|适用于所有导入，单位为s。|
|min_load_timeout_second|1|适用于所有导入，单位为s。|
|load_parallel_instance_num|1|单个BE上并发实例数，默认1个。|
|disable_hadoop_load|FALSE|是否禁用从Hadoop导入。取值为TRUE时禁用从Hadoop导入；取值为FALSE时不禁用。|
|disable_load_job|FALSE|如果集群异常时，是否接受导入任务。取值为TRUE时接受导入任务；取值为FALSE时不接受导入任务。|
|db_used_data_quota_update_interval_secs|300|更新数据库使用配额的时间周期，单位为s。|
|history_job_keep_max_second|604800|历史任务最大的保留时长，单位为s。|
|label_keep_max_num|1000|一定时间内所保留导入任务的最大数量，保留时间在label_keep_max_second中设置。|
|label_keep_max_second|259200|label保留时长，单位为s。|
|max_routine_load_job_num|100|最大的Routine Load作业数。|
|max_routine_load_task_concurrent_num|5|每个Routine Load作业最大并发执行的task数。|
|max_routine_load_task_num_per_be|5|每个BE最大并发执行的Routine Load task数，需小于等于BE的routine_load_thread_pool_size配置。|
|max_routine_load_batch_size|4294967296|每个Routine Load task导入的最大数据量。|
|routine_load_task_consume_second|15|每个Routine Load task消费数据的最大时间，单位为s。|
|routine_load_task_timeout_second|60|每个Routine Load task超时时间，单位为s。|
|max_tolerable_backend_down_num|0|如果故障的BE节点数超过该阈值，则不能自动恢复Routine Load作业。|
|period_of_auto_resume_min|5|自动恢复Routine Load的时间间隔。|
|spark_load_default_timeout_second|86400|Spark导入的超时时间，单位为s。|
|spark_home_default_dir|STARROCKS_HOME_DIR/lib/spark2x|Spark客户端根目录。|
|stream_load_default_timeout_second|600|StreamLoad超时时间，单位为s。|
|max_stream_load_timeout_second|259200|Stream导入的超时时间允许设置的最大值，单位为s。|
|insert_load_default_timeout_second|3600|Insert Into语句的超时时间，单位为s。|
|broker_load_default_timeout_second|14400|Broker Load的超时时间，单位为s。|
|min_bytes_per_broker_scanner|67108864|单个实例处理的最小数据量，默认64 MB。|
|max_broker_concurrency|100|单个任务最大并发实例数，默认100个。|
|export_max_bytes_per_be_per_task|268435456|单个导出任务在单个BE上导出的最大数据量，默认256 MB。|
|export_running_job_num_limit|5|导出作业最大的运行数目。|
|export_task_default_timeout_second|7200|导出作业超时时长，单位为s，默认2小时。|
|enable_strict_storage_medium_check|false|在创建表时，FE是否检查BE的存储介质类型。取值为true时检查；取值为false时不检查。|
|capacity_used_percent_high_water|0.75|Backend上磁盘使用容量的度量值。超过0.75之后，尽量不再往该tablet上发送建表和克隆的任务，直至恢复正常。|
|storage_high_watermark_usage_percent|85|BE存储目录下空间使用率的最大值。|
|storage_min_left_capacity_bytes|2147483648|BE存储目录下剩余空间的最小值，单位byte，默认2 GB。|
|storage_flood_stage_left_capacity_bytes|1073741824|BE存储目录的剩余空间。如果剩余空间小于该值，则会拒绝Load Restore作业，单位byte，默认1 GB。|
|storage_flood_stage_usage_percent|95|BE存储目录下空间使用率。如果空间使用率超过该值，则会拒绝Load和Restore作业。|
|catalog_trash_expire_second|86400|删表或数据库之后，元数据在回收站中保留的时长，单位为s，默认1天。超过该时长，则数据无法恢复。|
|alter_table_timeout_second|86400|Schema change超时时间，单位为s，默认1天。|
|balance_load_disk_safe_threshold|0.5|仅对disk_and_tablet策略有效。如果所有BE的磁盘使用率低于50%，则认为磁盘使用均衡。|
|balance_load_score_threshold|0.1|针对be_load_score策略，负载比平均负载低10%的BE处于低负载状态，比平均负载高10%的BE处于高负载状态。针对disk_and_tablet策略，如果最大和最小BE磁盘使用率之差高于10%，则认为磁盘使用不均衡，会触发tablet重新均衡。|
|disable_balance|false|是否禁用Tablet调度。取值为false时禁用；取值为true时不禁用。|
|max_scheduling_tablets|2000|可同时调度的tablet的数量。如果正在调度的tablet数量超过该值，则跳过tablet均衡检查。|
|max_balancing_tablets|100|正在均衡的tablet数量的最大值。如果正在均衡的tablet数量超过该值，则跳过tablet重新均衡。|
|disable_colocate_balance|false|是否禁用Colocate Table的副本均衡。取值为true时禁用；取值为false时不禁用。|
|recover_with_empty_tablet|false|在tablet副本丢失或损坏时，是否使用空的tablet代替。取值为true时使用空的tablet代替；取值为false时不使用空的tablet代替。使用空的tablet代替可保证在有tablet副本丢失或损坏时，query仍能执行，但结果可能错误。|
|min_clone_task_timeout_sec|180|克隆Tablet的最小超时时间，单位为s，默认3min。|
|max_clone_task_timeout_sec|7200|克隆Tablet的最大超时时间，单位为s，默认2h。|
|tablet_create_timeout_second|1|建表超时时长，单位为s。|
|tablet_delete_timeout_second|2|删除表的超时时间，单位为s。|
|tablet_repair_delay_factor_second|60|FE控制进行副本修复的间隔，单位为s。|
|consistency_check_start_time|23|FE发起副本一致性检测的起始时间，默认是23:00。|
|consistency_check_end_time|4|FE发起副本一致性检测的终止时间，默认是4:00。|
|check_consistency_default_timeout_second|600|副本一致性检测的超时时间，单位为s。|
|plugin_enable|TRUE|是否开启了插件功能。取值为TRUE时开启了插件功能；取值为FALSE时没有开启插件功能。只能在Master节点安装或卸载插件。|
|max_small_file_number|100|允许存储小文件数目的最大值。|
|max_small_file_size_bytes|1048576|存储文件的大小上限，单位byte，默认1 MB。|
|backup_job_default_timeout_ms|86400000|Backup作业的超时时间，单位为毫秒，默认1天。|
|report_queue_size|100|Disk、Task或Tablet的Report的等待队列长度。|

### FE静态参数
|参数|默认值|描述|
|---|---|---|
|log_roll_size_mb|1024|日志文件的大小，单位为MB。默认值1024表示每个日志文件的大小为1GB。|
|sys_log_dir|/opt/starrocks/be/log|存放日志的目录。|
|sys_log_level|INFO|系统日志的级别，可配置等级从宽松到严格依次为INFO、WARNING、ERROR和FATAL。|
|sys_log_verbose_modules|空字符串|日志打印的模块，如填写为org.apache.starrocks.catalog，则只打印catalog模块下的日志。|
|sys_log_roll_interval|DAY|系统日志滚动的时间间隔。|
|sys_log_delete_age|1d|系统日志文件的保留时长。|
|sys_log_roll_num|2|每个sys_log_roll_interval时间段内，允许保留的系统日志文件的最大数目。|
|audit_log_dir|starrocksFe.STARROCKS_HOME_DIR/log|审计日志保留的目录。|
|audit_log_roll_num|2|每个dump_log_roll_interval时间内，允许保留的Dump日志文件的最大数目。|
|audit_log_modules|slow_query, query|打印审计日志的模块，默认打印slow_query和query模块的日志，可指定多个模块，模块名称之间用英文逗号加一个空格分隔。|
|audit_log_roll_interval|DAY|审计日志滚动的时间间隔，取值为DAY和HOUR。|
|audit_log_delete_age|1d|审计日志文件的保留时长。|
|dump_log_dir|STARROCKS_HOME_DIR/log|Dump日志的目录。|
|dump_log_modules|query|打印Dump日志的模块，默认打印query模块的日志，可指定多个模块，模块名称之间用英文逗号加一个空格分隔。|
|dump_log_roll_interval|DAY|Dump日志拆分的时间间隔，日志文件的后缀为yyyyMMdd（DAY）或yyyyMMddHH（HOUR）。|
|dump_log_roll_num|2|每个dump_log_roll_interval时间内，保留的Dump日志文件的最大数目。|
|dump_log_delete_age|1d|Dump日志文件的保留时长。|
|frontend_address|0.0.0.0|FE节点的IP地址。|
|priority_networks|空字符串|以CIDR形式10.10.**.**/24指定IP地址，适用于机器有多个IP，需要指定优先使用的网络。|
|http_port|8030|Http Server的端口。|
|http_backlog_num|1024|HTTP Server的backlog队列长度。|
|cluster_name|StarRocks Cluster|Web页面中Title显示的集群名称。|
|rpc_port|9020|FE上的Thrift Server端口。|
|thrift_backlog_num|1024|Thrift Server的backlog队列长度。|
|thrift_server_type|THREAD_POOL|Thrift服务器的服务模型，取值范围：SIMPLE、THREADED和THREAD_POOL。|
|thrift_server_max_worker_threads|4096|Thrift Server最大工作线程数。|
|thrift_client_timeout_ms|5000|Thrift客户端链接的空闲超时时间，即链接超过该时间无新请求后则将链接断开。|
|brpc_idle_wait_max_time|10000|BRPC的空闲等待时间，单位为ms，默认为10s。|
|query_port|9030|FE上的MySQL Server端口。|
|mysql_service_nio_enabled|true|是否开启MySQL服务器的异步I/O选项。取值为true时开启；取值为false时不开启。|
|mysql_service_io_threads_num|4|FE连接服务线程数。|
|mysql_nio_backlog_num|1024|MySQL Server的backlog队列长度。|
|max_mysql_service_task_threads_num|4096|MySQL Server处理任务的最大线程数。|
|max_connection_scheduler_threads_num|4096|连接定时器的线程池的最大线程数。|
|qe_max_connection|1024|FE上最多接收的连接数，适用于所有用户。|
|check_java_version|true|检查已编译的Java版本与运行的Java版本是否兼容。如果不兼容，则上报Java版本不匹配的异常信息，并终止启动。|
|meta_dir|/opt/starrocks/fe/meta|元数据的保留目录。|
|heartbeat_mgr_threads_num|8|HeartbeatMgr中发送心跳任务的线程数。|
|heartbeat_mgr_blocking_queue_size|1024|HeartbeatMgr中发送心跳任务的线程池的队列长度。|
|metadata_failure_recovery|false|强制重置FE的元数据。（请谨慎使用该参数。）|
|edit_log_port|9010|FE Group（Master、Follower、Observer）之间通信用的端口。|
|edit_log_type|BDB|Edit log的类型，只能为BDB。|
|bdbje_heartbeat_timeout_second|30|BDBJE心跳超时的间隔，单位为s。|
|bdbje_lock_timeout_second|1|BDBJE锁超时的间隔，单位为s。|
|max_bdbje_clock_delta_ms|5000|Master与Non-master最大容忍的时钟偏移，单位为ms。|
|txn_rollback_limit|100|事务回滚的上限。|
|bdbje_replica_ack_timeout_second|10|BDBJE Master等待足够多的FOLLOWER ACK的最长时间。|
|master_sync_policy|SYNC|Master日志刷盘的方式，默认是SYNC。|
|replica_sync_policy|SYNC|Follower日志刷盘的方式，默认是SYNC。|
|meta_delay_toleration_second|300|非Master节点能够容忍的最大元数据落后的时间，单位为s。|
|cluster_id|-1|FE所在StarRocks实例的ID。具有相同集群ID的FE或BE属于同一个StarRocks实例。默认值 - 1，表示在Leader FE首次启动时随机生成一个。|
|disable_colocate_join|FALSE|是否开启Colocate Join。取值为FALSE时不开启；取值为TRUE时开启。|
|enable_udf|FALSE|是否开启UDF。取值为FALSE时不开启；取值为TRUE时开启。|
|publish_version_interval_ms|10|发送版本生效任务的时间间隔。|
|statistic_cache_columns|100000|缓存统计信息表的最大行数。|
|async_load_task_pool_size|10|导入任务执行的线程池大小。|
|load_checker_interval_second|5|导入轮询的间隔，单位为s。|
|transaction_clean_interval_second|30|清理已结束事务的周期，单位为s。|
|label_clean_interval_second|14400|label清理的间隔，单位为s。|
|spark_dpp_version|1.0.0|Spark dpp版本。|
|spark_resource_path|空字符串|Spark依赖包的根目录。|
|spark_launcher_log_dir|sys_log_dir/spark_launcher_log|Spark日志目录。|
|yarn_client_path|STARROCKS_HOME_DIR/lib/yarn-client/hadoop/bin/yarn|YARN客户端根目录。|
|yarn_config_dir|STARROCKS_HOME_DIR/lib/yarn-config|YARN配置文件目录。|
|export_checker_interval_second|5|导出线程轮询间隔，单位为s。|
|export_task_pool_size|5|导出任务线程池大小。|
|export_checker_interval_second|5|导出作业调度器的调度周期，单位为s。|
|tablet_sched_storage_cooldown_secon|-1|介质迁移的时间，单位为s。默认值 - 1表示不进行自动降冷。如需启用自动降冷功能，请显式设置参数取值大于0。|
|default_storage_medium|HDD|默认的存储介质，取值为HDD和SSD。在创建表或分区时，如果没有指定存储介质，则会使用该值。|
|schedule_slot_num_per_path|2|一个BE存储目录能够同时执行tablet相关任务的数目。|
|tablet_balancer_strategy|disk_and_tablet|Tablet均衡策略，取值为disk_and_tablet或be_load_score。|
|tablet_stat_update_interval_second|300|FE向每个BE请求收集tablet信息的时间间隔，单位为s，默认5min。|
|plugin_dir|STARROCKS_HOME_DIR/plugins|插件安装的目录。|
|small_file_dir|STARROCKS_HOME_DIR/small_files|小文件的根目录。|
|max_agent_task_threads_num|4096|代理任务的线程池的最大线程数。|
|authentication_ldap_simple_bind_base_dn|""|用户的base DN，指定用户的检索范围。|
|authentication_ldap_simple_bind_root_dn|""|检索用户时，使用的管理员账号DN。|
|authentication_ldap_simple_bind_root_pwd|""|检索用户时，使用的管理员账号密码。|
|authentication_ldap_simple_server_host|""|LDAP服务的host地址。|
|authentication_ldap_simple_server_port|389|LDAP服务的端口。|
|authentication_ldap_simple_user_search_attr|uid|LDAP对象中标识用户的属性名称。|
|tmp_dir|starrocksFe.STARROCKS_HOME_DIR/temp_ddir|临时文件保存目录，例如Backup和Restore等进程保留的目录。|
|locale|zh_CN.UTF-8|FE所使用的字符集。|
|hive_meta_load_concurrency|4|Hive元数据支持的最大并发线程数。|
|hive_meta_cache_refresh_interval_s|7200|定时刷新Hive外表元数据缓存的周期，单位为s。|
|hive_meta_cache_ttl_s|86400|Hive外表元数据缓存失效时间，单位为s，默认2h。|
|hive_meta_store_timeout_s|10|连接Hive MetaStore的超时时间，单位为s。|
|es_state_sync_interval_second|10|FE获取ElasticSearch Index的时间，单位为s。|
|enable_auth_check|true|是否开启鉴权。取值为true时开启鉴权；取值为false时不开启鉴权。|
|enable_metric_calculator|true|是否开启定期收集Metrics。取值为true时开启；取值为false时不开启。|

|配置项|默认值|描述|
|:--|:--|:--|
|be_port|9060|BE上Thrift Server的端口，用于接收来自FE的请求。|
|brpc_port|8060|BRPC的端口，可以查看BRPC的一些网络统计信息。|
|brpc_num_threads|-1|BRPC的bthreads线程数量。默认值 - 1表示和CPU核数一样。|
|priority_networks|空字符串|以CIDR形式10.10.**.**/24指定BE的IP地址，适用于机器有多个IP，需要指定优先使用的网络。|
|heartbeat_service_port|9050|心跳服务端口（Thrift），接收来自FE的心跳。|
|heartbeat_service_thread_count|1|心跳线程数。|
|create_tablet_worker_count|3|创建tablet的线程数。|
|drop_tablet_worker_count|3|删除tablet的线程数。|
|push_worker_count_normal_priority|3|导入线程数，处理NORMAL优先级任务。|
|push_worker_count_high_priority|3|导入线程数，处理HIGH优先级任务。|
|publish_version_worker_count|2|生效版本的线程数。|
|clear_transaction_task_worker_count|1|清理事务的线程数。|
|alter_tablet_worker_count|3|进行Schema Change的线程数。|
|clone_worker_count|3|克隆的线程数。|
|storage_medium_migrate_count|1|介质迁移的线程数。例如，热数据从SSD迁移到SATA盘的线程数。|
|check_consistency_worker_count|1|计算tablet的校验和checksum。|
|report_task_interval_seconds|10|汇报单个任务的间隔，单位为s。建表、删除表、导入和Schema Change都可以被认定是任务。|
|report_disk_state_interval_seconds|60|汇报磁盘状态的间隔，单位为s。汇报各个磁盘的状态及其数据量等。|
|report_tablet_interval_seconds|60|汇报tablet的间隔，单位为s。汇报所有的tablet的最新版本。|
|alter_tablet_timeout_seconds|86400|Schema Change超时时间，单位为s。|
|sys_log_dir|${DORIS_HOME}/log|存放日志的目录。日志级别包括INFO、WARNING、ERROR和FATAL。|
|user_function_dir|${DORIS_HOME}/lib/udf|存放UDF程序的目录。|
|sys_log_level|INFO|日志的等级。可以配置的等级从宽松到严格依次为INFO、WARNING、ERROR和FATAL。|
|sys_log_roll_mode|SIZE - MB - 1024|日志拆分的大小，每1GB拆分一个日志。|
|sys_log_roll_num|10|日志保留的数目。|
|sys_log_verbose_modules|空字符串|日志打印的模块。如果写olap，则只打印olap模块下的日志。|
|sys_log_verbose_level|10|日志显示的级别，用于控制代码中VLOG开头的日志输出。|
|log_buffer_level|空字符串|日志刷盘的策略，默认保持在内存中。|
|num_threads_per_core|3|每个CPU core启动的线程数。|
|compress_rowbatches|TRUE|BE之间RPC通信是否压缩RowBatch，用于查询层之间的数据传输。|
|serialize_batch|FALSE|BE之间RPC通信是否序列化RowBatch，用于查询层之间的数据传输。|
|status_report_interval|5|查询汇报profile的间隔，单位为s，用于FE收集查询统计信息。|
|doris_scanner_thread_pool_thread_num|48|存储引擎并发扫描磁盘的线程数，统一管理在线程池中。|
|doris_scanner_thread_pool_queue_size|102400|存储引擎最多接收的任务数。|
|doris_scan_range_row_count|524288|存储引擎拆分查询任务的粒度。|
|doris_scanner_queue_size|1024|存储引擎支持的扫描任务数。|
|doris_scanner_row_num|16384|每个扫描线程单次执行最多返回的数据行数。|
|doris_max_scan_key_num|1024|查询最多拆分的scan key数目。|
|column_dictionary_key_ratio_threshold|0|字符串类型的取值比例，小于这个比例采用字典压缩算法。|
|column_dictionary_key_size_threshold|0|字典压缩列大小，小于这个值采用字典压缩算法。|
|memory_limitation_per_thread_for_schema_change|2|单个Schema Change任务允许占用的最大内存。|
|file_descriptor_cache_clean_interval|3600|文件句柄缓存清理的间隔，单位为s，用于清理长期不用的文件句柄。|
|disk_stat_monitor_interval|5|磁盘状态检测的间隔，单位为s。|
|unused_rowset_monitor_interval|30|清理过期Rowset的时间间隔，单位为s。|
|storage_root_path|空字符串|存储数据的目录。|
|max_tablet_num_per_shard|1024|每个shard的tablet数目，用于划分tablet，防止单个目录下tablet子目录过多。|
|pending_data_expire_time_sec|1800|存储引擎保留的未生效数据的最大时长，单位为s。|
|inc_rowset_expired_sec|1800|在增量克隆场景下，已导入的数据，在存储引擎中保留的时间，单位为s。|
|max_garbage_sweep_interval|3600|磁盘进行垃圾清理的最大间隔，单位为s。|
|min_garbage_sweep_interval|180|磁盘进行垃圾清理的最小间隔，单位为s。|
|snapshot_expire_time_sec|172800|快照文件清理的间隔，单位为s，默认为48小时。|
|trash_file_expire_time_sec|259200|回收站清理的间隔，单位为s，默认为72小时。|
|file_descriptor_cache_capacity|16384|文件句柄缓存的容量。|
|min_file_descriptor_number|60000|BE进程的文件句柄limit要求的下限。|
|index_stream_cache_capacity|10737418240|BloomFilter、Min或Max等统计信息缓存的容量。|
|storage_page_cache_limit|0|PageCache的容量。|
|disable_storage_page_cache|TRUE|是否禁用Page Cache。取值为TRUE时禁用Page Cache；取值为FALSE时不禁用Page Cache。|
|base_compaction_check_interval_seconds|60|BaseCompaction线程轮询的间隔，单位为s。|
|base_compaction_num_threads_per_disk|1|每个磁盘BaseCompaction线程的数目。|
|base_cumulative_delta_ratio|0.3|BaseCompaction触发条件之一：Cumulative文件大小达到Base文件的比例。|
|base_compaction_interval_seconds_since_last_operation|86400|BaseCompaction触发条件之一：上一轮BaseCompaction距今的间隔。|
|cumulative_compaction_check_interval_seconds|1|CumulativeCompaction线程轮询的间隔，单位为s。|
|min_cumulative_compaction_num_singleton_deltas|5|CumulativeCompaction触发条件之一：Singleton文件数目要达到的下限。|
|max_cumulative_compaction_num_singleton_deltas|1000|CumulativeCompaction触发条件之一：Singleton文件数目要达到的上限。|
|cumulative_compaction_num_threads_per_disk|1|每个磁盘CumulativeCompaction线程的数目。|
|min_compaction_failure_interval_sec|120|Tablet Compaction失败之后，再次被调度的间隔，单位为s。|
|max_compaction_concurrency|-1|BaseCompaction + CumulativeCompaction的最大并发。默认值 - 1表示没有限制。|
|webserver_port|8040|Http Server端口。|
|webserver_num_workers|48|Http Server线程数。|
|periodic_counter_update_period_ms|500|Counter统计信息的间隔，单位为ms。|
|load_data_reserve_hours|4|小批量导入生成的文件保留的时间，单位为h。|
|load_error_log_reserve_hours|48|导入数据信息保留的时长，单位为h。|
|number_tablet_writer_threads|16|流式导入的线程数。|
|streaming_load_max_mb|10240|流式导入单个文件大小的上限。|
|streaming_load_rpc_max_alive_time_sec|1200|流式导入RPC的超时时间。|
|fragment_pool_thread_num|64|查询线程数，默认启动64个线程，后续查询请求动态创建线程。|
|fragment_pool_queue_size|2048|单节点上能够处理的查询请求上限。|
|enable_partitioned_aggregation|TRUE|是否使用PartitionAggregation。取值为TRUE时使用；取值为FALSE时不使用。|
|enable_token_check|TRUE|是否开启Token检验。取值为TRUE时开启Token检验；取值为FALSE时不开启。|
|load_process_max_memory_limit_bytes|107374182400|单节点上所有的导入线程占据的内存上限，默认为100GB。|
|load_process_max_memory_limit_percent|30|单节点上所有的导入线程占据的内存上限比例。|
|sync_tablet_meta|FALSE|存储引擎是否开sync保留到磁盘上。|
|thrift_rpc_timeout_ms|5000|Thrift超时的时长，单位为ms。|
|txn_commit_rpc_timeout_ms|10000|Txn超时的时长，单位为ms。|
|routine_load_thread_pool_size|10|例行导入的线程池数目。|
|tablet_meta_checkpoint_min_new_rowsets_num|10|TabletMeta Checkpoint的最小Rowset数目。|
|tablet_meta_checkpoint_min_interval_secs|600|TabletMeta Checkpoint线程轮询的时间间隔，单位为s。|
|brpc_max_body_size|209715200|BRPC最大的包容量，默认为200MB。|
|max_runnings_transactions|2000|存储引擎支持的最大事务数。|
|tablet_map_shard_size|32|Tablet分组数。|
|enable_bitmap_union_disk_format_with_set|FALSE|Bitmap新存储格式，可以优化bitmap_union性能。|

# be代码中的配置
## 代码相关配置

- `#pragma once`: 预处理指令，防止头文件被多次包含。
- `#include "configbase.h"`: 包含基础配置类的头文件。

## 配置名称及默认值
### be
- `cluster_id`: 集群ID，默认为"-1"。
- `be_port`: Backend服务端口，默认为"9060"。
- `thrift_port`: Thrift服务端口，默认为"0"。
- `brpc_port`: BRPC服务端口，默认为"8060"。
- `brpc_num_threads`: BRPC线程数，默认为CPU核心数。
- `brpc_max_connections_per_server`: BRPC客户端与每个服务器维护的最大单个连接数，默认为"1"。
- `priority_networks`: 服务器有多个IP时的选择策略，默认为空字符串。
- `net_use_ipv6_when_priority_networks_empty`: 当`priority_networks`为空时是否使用IPv6，默认为"false"。
- `enable_auto_adjust_pagecache`: 是否自动调整页缓存大小，默认为"true"。
- `memory_urgent_level`: 内存紧急水位线，默认为"85"。
- `memory_high_level`: 内存高水位线，默认为"75"。
- `pagecache_adjust_period`: 页缓存大小调整周期，默认为"20"。
- `auto_adjust_pagecache_interval_seconds`: 页缓存调整迭代间隔时间，默认为"10"。
- `mem_limit`: 进程内存限制，默认为"90%"。
- `enable_jemalloc_memory_tracker`: 是否启用jemalloc内存跟踪，默认为"true"。
- `jemalloc_fragmentation_ratio`: 认为jemalloc内存的一部分为碎片的比例，默认为"0.3"。
- `heartbeat_service_port`: 心跳服务端口，默认为"9050"。
- `heartbeat_service_thread_count`: 心跳服务线程数，默认为"1"。
- `create_tablet_worker_count`: 创建表线程数，默认为"3"。
- `drop_tablet_worker_count`: 删除表线程数，默认为"3"。
- `push_worker_count_normal_priority`: 普通优先级批量加载线程数，默认为"3"。
- `push_worker_count_high_priority`: 高优先级批量加载线程数，默认为"3"。
- `transaction_publish_version_worker_count`: 发布版本事务线程数，默认为"0"。
- `transaction_publish_version_thread_pool_num_min`: 发布版本事务最小线程池数，默认为"0"。
- `transaction_apply_worker_count`: 应用行集到主键表的线程数，默认为CPU核心数。
- `get_pindex_worker_count`: 获取主键索引工作线程数，默认为"0"。
- `transaction_apply_thread_pool_num_min`: 应用事务最小线程池数，默认为"0"。
- `transaction_apply_worker_idle_time_ms`: 事务应用工作线程空闲时间，默认为"500"。
- `clear_transaction_task_worker_count`: 清除事务任务线程数，默认为"1"。
- `delete_worker_count_normal_priority`: 普通优先级删除线程数，默认为"2"。
- `delete_worker_count_high_priority`: 高优先级删除线程数，默认为"1"。
- `alter_tablet_worker_count`: 更改表线程数，默认为"3"。
- `parallel_clone_task_per_path`: 每个存储路径的并行克隆任务数，默认为"8"。
- `clone_worker_count`: 克隆线程数，默认为"3"。
- `storage_medium_migrate_count`: 存储介质迁移线程数，默认为"3"。
- `check_consistency_worker_count`: 检查一致性线程数，默认为"1"。
- `update_schema_worker_count`: 更新模式线程数，默认为"3"。
- `upload_worker_count`: 上传线程数，默认为"1"。
- `download_worker_count`: 下载线程数，默认为"1"。
- `make_snapshot_worker_count`: 制作快照线程数，默认为"5"。
- `release_snapshot_worker_count`: 释放快照线程数，默认为"5"。
- `report_task_interval_seconds`: 报告任务签名到FE的间隔时间，默认为"10"。
- `report_disk_state_interval_seconds`: 报告磁盘状态到FE的间隔时间，默认为"60"。
- `report_tablet_interval_seconds`: 报告OLAP表到FE的间隔时间，默认为"60"。
- `report_workgroup_interval_seconds`: 报告工作组到FE的间隔时间，默认为"5"。
- `report_resource_usage_interval_ms`: 报告资源使用情况到FE的间隔时间，默认为"1000"。
- `max_download_speed_kbps`: 最大下载速度，默认为"50000"。
- `download_low_speed_limit_kbps`: 下载低速限制，默认为"50"。
- `download_low_speed_time`: 下载低速时间，默认为"300"。
- `sleep_one_second`: 一秒钟睡眠时间，默认为"1"。
- `sleep_five_seconds`: 五秒钟睡眠时间，默认为"5"。
- `compact_threads`: 压缩线程数，默认为"4"。
- `compact_thread_pool_queue_size`: 压缩线程池队列大小，默认为"100"。
- `replication_threads`: 复制线程数，默认为"0"。
- `replication_max_speed_limit_kbps`: 复制最大速度限制，默认为"50000"。
- `replication_min_speed_limit_kbps`: 复制最小速度限制，默认为"50"。
- `replication_min_speed_time_seconds`: 复制最小速度时间，默认为"300"。
- `clear_expired_replication_snapshots_interval_seconds`: 清除过期复制快照间隔时间，默认为"3600"。
- `sys_log_dir`: 系统日志目录，默认为"${STARROCKS_HOME}/log"。
- `user_function_dir`: 用户函数目录，默认为"${STARROCKS_HOME}/lib/udf"。
- `sys_log_level`: 系统日志级别，默认为"INFO"。
- `sys_log_roll_mode`: 系统日志滚动模式，默认为"SIZE-MB-1024"。
- `sys_log_roll_num`: 日志滚动数，默认为"10"。
- `sys_log_verbose_modules`: 详细日志模块，默认为空字符串。
- `sys_log_verbose_level`: 详细日志级别，默认为"10"。
- `log_buffer_level`: 日志缓冲级别，默认为空字符串。
- `pull_load_task_dir`: 拉取加载任务目录，默认为"${STARROCKS_HOME}/var/pull_load"。
- `web_log_bytes`: Web服务器日志页面的最大字节数，默认为"1048576"。
- `be_service_threads`: 用于服务后端执行请求的线程数，默认为"64"。
- `default_query_options`: StarRocks的默认查询选项，默认为空字符串。
- `num_threads_per_core`: 每个核心的工作线程数，默认为"3"。
- `compress_rowbatches`: 是否压缩行批次数据，默认为"true"。
- `rpc_compress_ratio_threshold`: 网络中行批次压缩比率阈值，默认为"1.1"。
- `serialize_batch`: 是否序列化每个返回的行批次，默认为"false"。
- `status_report_interval`: 状态报告间隔时间，默认为"5"。
- `local_library_dir`: 从HDFS复制UDF库到本地的目录，默认为"${UDF_RUNTIME_DIR}"。
- `scanner_thread_pool_thread_num`: OLAP/外部扫描器线程池大小，默认为"48"。
- `scanner_thread_pool_queue_size`: OLAP/外部扫描器线程池队列大小，默认为"102400"。
- `udf_thread_pool_size`: UDF线程池大小，默认为"1"。
- `port`: 用于运行StarRocks测试后端的端口，默认为"20001"。
- `thrift_connect_timeout_seconds`: Thrift客户端连接超时时间，默认为"3"。
- `broker_load_default_timeout_second`: Broker加载的默认超时时间，默认为"14400"。
- `broker_load_min_chunk_size`: Broker加载的最小数据块大小，默认为"100"。
- `broker_load_max_memory_limit`: Broker加载的最大内存限制，默认为"2147483648"（即2GB）。
- `load_pending_thread_num_high_priority`: 高优先级加载等待线程数，默认为"3"。
- `load_pending_thread_num_normal_priority`: 普通优先级加载等待线程数，默认为"10"。
- `load_running_thread_num_high_priority`: 高优先级加载运行线程数，默认为"10"。
- `load_running_thread_num_normal_priority`: 普通优先级加载运行线程数，默认为"25"。
- `max_unfinished_load_job`: 最大未完成加载作业数，默认为"100"。
- `stream_load_default_timeout_second`: Stream加载的默认超时时间，默认为"300"。
- `max_stream_load_timeout_second`: Stream加载的最大超时时间，默认为"259200"（即3天）。
- `stream_load_rpc_max_alive_time_sec`: Stream加载RPC的最大存活时间，默认为"600"。
- `max_routine_load_job_num`: 最大例行加载作业数，默认为"100"。
- `routine_load_task_timeout_second`: 例行加载任务的超时时间，默认为"10"。
- `routine_load_task_consume_second`: 例行加载任务的消耗时间，默认为"3"。
- `routine_load_scheduler_interval_second`: 例行加载调度器的间隔时间，默认为"5"。
- `max_routine_load_task_num_per_be`: 每个BE的最大例行加载任务数，默认为"5"。
- `max_small_file_size_bytes`: 最大小文件大小，默认为"1048576"（即1MB）。
- `max_small_file_number`: 最大小文件数量，默认为"100"。
- `max_routine_load_batch_size`: 最大例行加载批次大小，默认为"209715200"（即200MB）。
- `routine_load_thread_pool_size`: 例行加载线程池大小，默认为"10"。
- `routine_load_task_concurrent_num`: 例行加载任务并发数，默认为"3"。
- `max_routine_load_task_exec_interval_ms`: 最大例行加载任务执行间隔时间，默认为"10000"（即10秒）。
- `min_routine_load_task_exec_interval_ms`: 最小例行加载任务执行间隔时间，默认为"1000"（即1秒）。
- `routine_load_task_retry_delay_ms`: 例行加载任务重试延迟时间，默认为"10000"（即10秒）。
- `max_routine_load_task_retry_num`: 最大例行加载任务重试次数，默认为"3"。
- `routine_load_kafka_default_max_batch_interval_ms`: 例行加载Kafka默认最大批次间隔时间，默认为"100"。
- `routine_load_kafka_default_max_batch_rows`: 例行加载Kafka默认最大批次行数，默认为"10000"。
- `routine_load_kafka_default_max_batch_size`: 例行加载Kafka默认最大批次大小，默认为"1048576"（即1MB）。
- `routine_load_kafka_default_max_error_number`: 例行加载Kafka默认最大错误数，默认为"10000"。
- `routine_load_kafka_default_poll_timeout_ms`: 例行加载Kafka默认轮询超时时间，默认为"100"。
- `routine_load_kafka_max_partitions`: 例行加载Kafka最大分区数，默认为"100"。
- `max_routine_load_kafka_single_message_bytes`: 最大例行加载Kafka单条消息字节数，默认为"1048576"（即1MB）。
- `max_routine_load_kafka_total_message_bytes`: 最大例行加载Kafka总消息字节数，默认为"104857600"（即100MB）。
- `max_routine_load_kafka_error_rate`: 最大例行加载Kafka错误率，默认为"0.1"。
- `routine_load_kafka_progress_sync_interval_ms`: 例行加载Kafka进度同步间隔时间，默认为"10000"（即10秒）。
- `max_routine_load_kafka_consume_speed`: 最大例行加载Kafka消费速度，默认为"1000000"（即1MB/s）。
- `routine_load_kafka_default_max_offset_lag`: 例行加载Kafka默认最大偏移滞后，默认为"100000"。
- `max_routine_load_kafka_partition_offset_lag`: 最大例行加载Kafka分区偏移滞后，默认为"100000000"。

### 删除数据及磁盘回收站管理
- `catalog_trash_expire_second`:删除表/数据库之后，元数据在回收站中保留的时长，超过这个时长，数据就不可以通过RECOVER 语句恢复。默认为 86400s。
- `path_scan_interval_second`: GC 线程清理过期数据的间隔时间.默认为 86400s。
- `trash_file_expire_time_sec`: 回收站清理的间隔。自 v2.5.17、v3.0.9 以及 v3.1.6 起，默认值由 259200 变为 86400s。
- `max_garbage_sweep_interval`: 磁盘进行垃圾清理的最大间隔。自 3.0 版本起，该参数由静态变为动态。3600s。

