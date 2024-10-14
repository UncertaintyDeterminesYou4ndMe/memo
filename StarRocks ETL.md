
### 加工示例
```sql



##demo ETL 加工
--建表
--drop table data_output.ads_lzb_activity_reward_distribution_report
CREATE TABLE IF NOT EXISTS `data_output`.`ads_lzb_activity_reward_distribution_report` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `trans_id` VARCHAR(255) NOT NULL COMMENT '订单流水id',
  `user_id` VARCHAR(255) NOT NULL COMMENT '用户userId',
  `biz_type` VARCHAR(255) NOT NULL COMMENT '订单类型',
  `title` VARCHAR(255) NOT NULL DEFAULT '金币兑换钱包' COMMENT '任务名称',
  `biz_merchant` VARCHAR(50) NOT NULL DEFAULT '来助宝' COMMENT '业务归属',
  `cash_num` bigint NOT NULL COMMENT '金额',
  `gmt_created` DATETIME NOT NULL COMMENT '日期'
) 
DUPLICATE KEY(id, trans_id)
COMMENT "来助宝CPA任务报表"
PARTITION BY date_trunc('day', gmt_created);
-- 初始化历史存量数据时使用 insert into ，历史数据会落到分区上。增量加工时使用 insert overwrite partition 写入对应分区增量的数据（overwrite 保证可以重跑幂等性）
insert into data_output.ads_lzb_activity_reward_distribution_report ( 
    trans_id,
    user_id,
    biz_type,
    title,
    biz_merchant,
    cash_num,
    gmt_created
)
SELECT
    ucat.trans_id AS 订单流水id,
    ucat.user_id AS 用户id,
    CASE
        WHEN ucat.biz_type = 'CPA' THEN '活动奖励'
        WHEN ucat.biz_type = 'EXCHANGE' THEN '金币兑换'
    END AS 订单类型,
    COALESCE(co.title, '金币兑换钱包') AS 任务名称,
    '来助宝' AS 业务归属,
    ucat.cash_num AS 金额,
    ucat.gmt_created AS 时间
FROM
    whale.user_cash_account_trans ucat
LEFT JOIN
    whale.cpa_order co
ON
    ucat.out_biz_num = co.order_id
    AND ucat.biz_type = 'CPA'
    AND co.task_type != 'CPA'
WHERE
    ucat.opt_type = 'ADD'
AND
    ucat.gmt_modified >= '2024-09-01' and ucat.gmt_modified < '2024-10-01'
AND (
        (ucat.biz_type = 'CPA' AND ucat.out_biz_num = co.order_id AND co.task_type != 'CPA')
        OR ucat.biz_type = 'EXCHANGE'
    )
;
select * from data_output.ads_lzb_activity_reward_distribution_report 
select count(*) from data_output.ads_lzb_activity_reward_distribution_report 
SHOW PARTITIONS FROM data_output.ads_lzb_activity_reward_distribution_report 







##demo ETL 加工（物化视图）
show materialized views from data_output WHERE NAME = 'ads_mv_user_zl_count_stat';

CREATE MATERIALIZED VIEW IF NOT EXISTS data_output.ads_mv_user_zl_count_stat
COMMENT "用户助力总次数统计视图"
DISTRIBUTED BY HASH(user_id, demand_type)
REFRESH ASYNC EVERY (INTERVAL 5 MINUTE)
AS
select 
     user_id, 
     demand_type, 
     count(*) assist_count
from 
    benefit.crowd_source_user_finish_record
group BY
     user_id, demand_type
;

select count(*)
from 
(
     select 
     user_id, 
     demand_type, 
     count(*) assist_count
from 
    benefit.crowd_source_user_finish_record
group BY
     user_id, demand_type
) ta
;

select * from data_output.ads_mv_user_zl_count_stat;

select count(*) from benefit.crowd_source_user_finish_record

select count(*) from data_output.ads_mv_user_zl_count_stat; --343228





```
