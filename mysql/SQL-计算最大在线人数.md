# SQL讲解 —— 计算直播间最大在线人数

### 表格

有用户观看直播记录表：user_id为用户id，live_id为直播间id，in_time和out_time分别为用户进入和退出直播间的时间

| user_id | live_id | in_time             | out_time            |
| ------- | ------- | ------------------- | ------------------- |
| 0001    | 1       | 2023-01-01 10:00:00 | 2023-01-01 11:00:00 |
| 0002    | 2       | 2023-01-01 15:00:00 | 2023-01-01 16:30:00 |
| 0003    | 1       | 2023-01-01 10:20:00 | 2023-01-01 10:30:00 |
| ...     | ...     | ...                 | ...                 |

### 问题

求各场直播的最大在线人数。

| 直播间 | 最大在线人数 |
| ------ | ------------ |
| ？     | ？           |

### 分析

详细见PPT

1：把原表的时间区间转换为时间点，并设定in为1，out为-1（使用union all)

2：统计每个时间点的在线人数（使用窗口函数)

3：以直播间为分组，求出最大在线人数（使用group by)

### 解答

```sql
# 对应第一步
with t1 as (
    SELECT live_id,user_id,in_time dt,1 AS cnt FROM SQL_13
    UNION ALL
    SELECT live_id,user_id,out_time dt,-1 AS cnt FROM SQL_13
),
# 对应第二步
t2 as (
    SELECT live_id, user_id, dt, cnt, SUM(cnt) OVER(PARTITION BY live_id ORDER BY dt ASC) online_num FROM t1
)
# 对应第三步
SELECT live_id, MAX(online_num) as max_online_num FROM t2
GROUP BY live_id
ORDER BY live_id asc;
```

### 脚本

```sql
DROP TABLE if exists SQL_13;
CREATE TABLE SQL_13(
   user_id INT,
   live_id INT,
   in_time DATETIME NOT NULL,
   out_time DATETIME NOT NULL
);
insert into SQL_13(user_id,live_id,in_time,out_time)
values (0001,1,'2023-01-01 10:00:00','2023-01-01 11:00:00');
insert into SQL_13(user_id,live_id,in_time,out_time)
values (0002,2,'2023-01-01 15:00:00','2023-01-01 16:30:00');
insert into SQL_13(user_id,live_id,in_time,out_time)
values (0003,1,'2023-01-01 10:20:00','2023-01-01 10:30:00');
insert into SQL_13(user_id,live_id,in_time,out_time)
values (0004,1,'2023-01-01 09:20:00','2023-01-01 11:30:00');
insert into SQL_13(user_id,live_id,in_time,out_time)
values (0005,2,'2023-01-01 11:20:00','2023-01-01 14:30:00');
insert into SQL_13(user_id,live_id,in_time,out_time)
values (0006,2,'2023-01-01 12:20:00','2023-01-01 15:30:00');
insert into SQL_13(user_id,live_id,in_time,out_time)
values (0007,1,'2023-01-01 08:20:00','2023-01-01 09:30:00');
insert into SQL_13(user_id,live_id,in_time,out_time)
values (0008,2,'2023-01-01 08:20:00','2023-01-01 12:30:00');
insert into SQL_13(user_id,live_id,in_time,out_time)
values (0009,1,'2023-01-01 15:20:00','2023-01-01 19:30:00');
insert into SQL_13(user_id,live_id,in_time,out_time)
values (0010,2,'2023-01-01 09:20:00','2023-01-01 13:30:00');
select * from SQL_13;
```


