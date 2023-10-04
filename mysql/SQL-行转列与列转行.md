# SQL 讲解 —— 行转列 与 列转行

## 行转列

### 题目1

#### 描述

```
name  subject  score
张三    语文     78
张三    数学     88
张三    英语 	 98
李四    语文     89
李四    数学     76
李四    英语     90
王五    语文	 99
王五	  数学	 66
王五	  英语	 91

name  语文  数学  英语
张三   78   88    98
李四   89   76    90
王五   99   66    91
```



#### 脚本

```sql
create table SQL_1
(
    name varchar(20),
    subject varchar(20),
    score float
);
insert into SQL_1 (name, subject, score) values ('张三', '语文', 78);
insert into SQL_1 (name, subject, score) values ('张三', '数学', 88);
insert into SQL_1 (name, subject, score) values ('张三', '英语', 98);
insert into SQL_1 (name, subject, score) values ('李四', '语文', 89);
insert into SQL_1 (name, subject, score) values ('李四', '数学', 76);
insert into SQL_1 (name, subject, score) values ('李四', '英语', 90);
insert into SQL_1 (name, subject, score) values ('王五', '语文', 99);
insert into SQL_1 (name, subject, score) values ('王五', '数学', 66);
insert into SQL_1 (name, subject, score) values ('王五', '英语', 91);
select * from SQL_1;
```



#### 解题步骤

1） 确定分组列，转换列，数据列

2） 生成伪列

3） 做分组查询

4） 选择合适的聚合函数



#### 两步法

##### 公式：

```sql
select 分组列,
       聚合函数(m1) as 列名1,
       聚合函数(m2) as 列名2,
       聚合函数(m3) as 列名3
from (select *,
             case 转换列 when 转换列值1 then 数据列 else ... end as m1,
             case 转换列 when 转换列值2 then 数据列 else ... end as m2,
             case 转换列 when 转换列值3 then 数据列 else ... end as m3
      from 表名) 临时表名
group by 分组列;
```

##### 解题SQL

```sql
select name,
       sum(m1) as 语文,
       sum(m2) as 数学,
       sum(m3) as 英语
from (select *,
             case subject when '语文' then score else 0 end as m1,
             case subject when '数学' then score else 0 end as m2,
             case subject when '英语' then score else 0 end as m3
      from sql_1) tmp
group by name;
```



#### 一步法

##### 公式：

```sql
select 分组列，
       聚合函数(case 转换列 when 转换列值1 then 数据列 else ... end) as 列名1,
	   聚合函数(case 转换列 when 转换列值2 then 数据列 else ... end) as 列名2,
	   聚合函数(case 转换列 when 转换列值3 then 数据列 else ... end) as 列名3
	   ...
from 表名
group by 分组列;

select 分组列，
       聚合函数(case when 转换列=转换列值1 then 数据列 else ... end) as 列名1,
	   聚合函数(case when 转换列=转换列值2 then 数据列 else ... end) as 列名2,
	   聚合函数(case when 转换列=转换列值3 then 数据列 else ... end) as 列名3
	   ...
from 表名
group by 分组列;
```

##### 解题SQL

```sql
select name,
       sum(case subject when '语文' then score else 0 end) as 语文,
       sum(case subject when '数学' then score else 0 end) as 数学,
       sum(case subject when '英语' then score else 0 end) as 英语
from sql_1
group by name;

select name,
       sum(case when subject = '语文' then score else 0 end) as 语文,
       sum(case when subject = '数学' then score else 0 end) as 数学,
       sum(case when subject = '英语' then score else 0 end) as 英语
from sql_1
group by name;
```



### 题目2

#### 描述

```txt
# 日期        结果
# 2022-01-01  胜
# 2022-01-01  胜
# 2022-01-02  负
# 2022-01-02  负
# 2022-01-01  负
# 2022-01-02  负
# 2022-01-02  胜

# 日期         胜 负
# 2022-01-01  2  1
# 2022-01-02  1  3
```

#### 脚本

```sql
create table SQL_2(
   ddate varchar(10), result varchar(2)
);

insert into SQL_2 (ddate, result) values('2022-01-01','胜');
insert into SQL_2 (ddate, result) values('2022-01-01','胜');
insert into SQL_2 (ddate, result) values('2022-01-02','负');
insert into SQL_2 (ddate, result) values('2022-01-02','负');
insert into SQL_2 (ddate, result) values('2022-01-01','负');
insert into SQL_2 (ddate, result) values('2022-01-02','负');
insert into SQL_2 (ddate, result) values('2022-01-02','胜');
select * from SQL_2;
```

#### 解题SQL

```sql
select ddate,
       sum(case when result = '胜' then 1 else 0 end) as 胜,
       sum(case when result = '负' then 1 else 0 end) as 负
from sql_2
group by ddate;
```



## 列转行

### 题目3

#### 描述

```txt
name  语文  数学  英语 
张三   78   88    98
李四   89   76    90
王五   99   66    91 

name  subject  score
张三    语文     78
张三    数学     88
张三    英语 	 98
李四    语文     89
李四    数学     76
李四    英语     90
王五    语文	 99
王五	  数学	 66
王五	  英语	 91
```



#### 脚本

```sql
CREATE TABLE SQL_3 (
	name varchar(20),
    语文 float,
    数学 float,
    英语 float
);

insert into SQL_3 (name, '语文', '数学', '英语') values ('张三', 78, 88, 98);
insert into SQL_3 (name, '语文', '数学', '英语') values ('李四', 89, 76, 90);
insert into SQL_3 (name, '语文', '数学', '英语') values ('王五', 99, 66, 91);
```



#### 解题步骤

1） 确定转换列，非转换列

2） 生成新列

3） 使用union或union all 进行合并

4） 根据需要进行order by



#### 公式

```sql
SELECT 非转换列, '转换列1' AS 新转换列名, 转换列1 AS 新数据列名 FROM 表名
UNION ALL
SELECT 非转换列, '转换列2' AS 新转换列名, 转换列2 AS 新数据列名 FROM 表名
UNION ALL
SELECT 非转换列, '转换列3' AS 新转换列名, 转换列3 AS 新数据列名 FROM 表名
ORDER BY ...;
```



#### 解题SQL

```sql
SELECT name,'语文' AS subject,语文 AS score FROM SQL_3
UNION ALL
SELECT name,'数学' AS subject,数学 AS score FROM SQL_3
UNION ALL
SELECT name,'英语' AS subject,英语 AS score FROM SQL_3
ORDER BY name ASC, subject DESC;
```



### 题目4

#### 描述

```txt
Q1     Q2     Q3     Q4
1000   2000   3000   4000

季度   业绩
Q1    1000
Q2    2000
Q3    3000
Q4    4000
```



#### 脚本

```sql
CREATE TABLE SQL_4 (
	Q1 int, Q2 int, Q3 int, Q4 int
);

insert into SQL_4 values (1000, 2000, 3000, 4000);
```



#### 解题SQL

```sql
SELECT  'Q1' AS 季度, Q1 AS 业绩 FROM SQL_4
UNION ALL
SELECT 'Q2' AS 季度, Q2 AS 业绩 FROM SQL_4
UNION ALL
SELECT 'Q3' AS 季度, Q3 AS 业绩 FROM SQL_4
UNION ALL
SELECT 'Q4' AS 季度, Q4 AS 业绩 FROM SQL_4
ORDER BY 季度;
```



### 技巧：

扩展列：select ... as 新列名

减少列：直接不写

扩展行：union/ union all

减少行:  聚合函数



