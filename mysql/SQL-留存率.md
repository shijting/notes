# SQL讲解 —— 留存率

## 一. 留存率业务含义

留存率可以评用户对产品的粘性，留存率越低表明用户对产品粘性越小

留存率通常分为次日留存率、3日留存率、7日留存率、30日留存率

这里以新增用户留存率为例：

> **次日留存率：**基准日之后的第1天留存的用户数）/基准日当天新增用户数；
> **第3日留存率：**基准日之后的第3天留存的用户数）/基准日当天新增用户数；
> **第7日留存率：**基准日之后的第7天留存的用户数）/基准日当天新增用户数；
> **第30日留存率：**基准日之后的第30天留存的用户数）/基准日当天新增用户数；
>
> 留存率=各日留存用户数/基准日新增用户数



## 二. 相关题目

### 题目一

#### 表格

 SQL_11_REG (用户注册表)

| uid  |    register_time    |
| :--: | :-----------------: |
|  1   | 2020-01-01 00:01:00 |
|  2   | 2020-01-01 00:01:00 |
|  3   | 2020-01-01 00:01:00 |
|  4   | 2020-01-01 00:01:00 |
| ...  |         ...         |

 SQL_11_LOGIN (用户登录表)

| id   | uid  | login_time          |
| ---- | ---- | ------------------- |
| 1    | 1    | 2020-01-02 00:02:00 |
| 2    | 2    | 2020-01-02 00:02:00 |
| 3    | 3    | 2020-01-02 00:02:00 |
| 4    | 1    | 2020-01-03 00:02:00 |
| 5    | 3    | 2020-01-03 00:02:00 |
| ...  | ...  | ...                 |

#### 脚本

```sql
DROP TABLE if exists SQL_11_REG;
CREATE TABLE SQL_11_REG(
    uid INT AUTO_INCREMENT PRIMARY KEY,
    register_time DATETIME NOT NULL
);
insert into SQL_11_REG(register_time) values ('2020-01-01 00:01:00');
insert into SQL_11_REG(register_time) values ('2020-01-01 00:01:00');
insert into SQL_11_REG(register_time) values ('2020-01-01 00:01:00');
insert into SQL_11_REG(register_time) values ('2020-01-01 00:01:00');
insert into SQL_11_REG(register_time) values ('2020-01-02 00:01:00');
insert into SQL_11_REG(register_time) values ('2020-01-02 00:01:00');
insert into SQL_11_REG(register_time) values ('2020-01-02 00:01:00');
insert into SQL_11_REG(register_time) values ('2020-01-03 00:01:00');
insert into SQL_11_REG(register_time) values ('2020-01-03 00:01:00');
insert into SQL_11_REG(register_time) values ('2020-01-03 00:01:00');
insert into SQL_11_REG(register_time) values ('2020-01-03 00:01:00');
insert into SQL_11_REG(register_time) values ('2020-01-03 00:01:00');
select * from SQL_11_REG;

DROP TABLE if exists SQL_11_LOGIN;
CREATE TABLE SQL_11_LOGIN(
     id INT AUTO_INCREMENT PRIMARY KEY,
     uid INT NOT NULL,
     login_time DATETIME NOT NULL
);
insert into SQL_11_LOGIN(uid, login_time) values (1, '2020-01-02 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (2, '2020-01-02 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (3, '2020-01-02 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (1, '2020-01-03 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (3, '2020-01-03 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (4, '2020-01-03 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (6, '2020-01-03 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (2, '2020-01-04 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (3, '2020-01-04 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (5, '2020-01-04 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (6, '2020-01-04 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (7, '2020-01-04 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (9, '2020-01-04 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (1, '2020-01-05 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (6, '2020-01-05 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (7, '2020-01-05 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (9, '2020-01-05 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (10, '2020-01-05 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (11, '2020-01-05 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (4, '2020-01-06 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (11, '2020-01-06 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (12, '2020-01-06 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (1, '2020-01-07 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (2, '2020-01-07 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (6, '2020-01-07 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (8, '2020-01-07 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (9, '2020-01-07 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (2, '2020-01-08 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (10, '2020-01-08 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (12, '2020-01-08 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (3, '2020-01-09 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (8, '2020-01-09 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (9, '2020-01-09 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (11, '2020-01-09 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (4, '2020-01-10 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (6, '2020-01-10 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (7, '2020-01-10 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (9, '2020-01-10 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (10, '2020-01-10 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (12, '2020-01-10 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (6, '2020-01-11 00:02:00');
insert into SQL_11_LOGIN(uid, login_time) values (12, '2020-01-11 00:02:00');
select * from SQL_11_LOGIN;
```

#### 问题一

计算1月2日的次日留存率

```sql
SELECT
    count(DISTINCT l.uid)/count(DISTINCT r.uid) rr1
FROM SQL_11_REG r 
        LEFT JOIN SQL_11_LOGIN l ON (l.uid = r.uid AND date(l.login_time) = date(r.register_time) + INTERVAL '1' DAY)
WHERE date(r.register_time) = '2020-01-02';
```

日期时间间隔的多种表示方法

```sql
-- date1 = date2 + interval N unit
date(l.login_time) = date(r.register_time) + INTERVAL '1' DAY

-- date_sub/date_add
date_sub(date(l.login_time) ,INTERVAL '1' DAY) = date(r.register_time)

-- datediff
datediff(date(l.login_time),date(r.register_time)) = 1
```

#### 问题二

计算所有注册日的次日留存率

```sql
SELECT date(r.register_time) reg_date,
       count(DISTINCT l1.uid)/count(DISTINCT r.uid) rr1
FROM SQL_11_REG r 
         LEFT JOIN SQL_11_LOGIN l1 ON (l1.uid = r.uid AND date(l1.login_time) = date(r.register_time) + INTERVAL '1' DAY)
GROUP BY date(r.register_time);
```

#### 问题三

计算所有注册日的次日留存率，3日留存率，7日留存率

```sql
SELECT date(r.register_time) reg_date,
       count(DISTINCT l1.uid)/count(DISTINCT r.uid) rr1,
       count(DISTINCT l2.uid)/count(DISTINCT r.uid) rr3,
       count(DISTINCT l3.uid)/count(DISTINCT r.uid) rr7
FROM SQL_11_REG r
         LEFT JOIN SQL_11_LOGIN l1 ON (l1.uid = r.uid AND date(l1.login_time) = date(r.register_time) + INTERVAL '1' DAY)
         LEFT JOIN SQL_11_LOGIN l2 ON (l2.uid = r.uid AND date(l2.login_time) = date(r.register_time) + INTERVAL '3' DAY)
         LEFT JOIN SQL_11_LOGIN l3 ON (l3.uid = r.uid AND date(l3.login_time) = date(r.register_time) + INTERVAL '7' DAY)
GROUP BY date(r.register_time);
```

#### 优化 (减少join次数)

```sql
with t1 as (
    SELECT r.uid id,  date(r.register_time) reg_date, date(l.login_time) login_date, datediff(date(l.login_time),date(r.register_time)) diff
    FROM SQL_11_REG r LEFT JOIN SQL_11_LOGIN l
                 ON l.uid = r.uid AND date(l.login_time) BETWEEN date(r.register_time) + INTERVAL '1' DAY AND date(r.register_time) + INTERVAL '7' DAY
)
select reg_date,
       COUNT(DISTINCT CASE WHEN diff=1 THEN id END)/COUNT(DISTINCT id) rr1,
       COUNT(DISTINCT CASE WHEN diff=3 THEN id END)/COUNT(DISTINCT id) rr3,
       COUNT(DISTINCT CASE WHEN diff=7 THEN id END)/COUNT(DISTINCT id) rr7
from t1 group by reg_date;
```

#### 公式

```
with 临时表 as (
    SELECT a.id id, a.基准时间 , b.留存时间, datediff(b.留存时间,a.基准时间) '留存间隔'
    FROM 基准表 a LEFT JOIN 留存表 b
        ON b.关联id = a.id AND b.留存时间 BETWEEN a.基准时间 + INTERVAL '1' DAY AND a.基准时间 + INTERVAL '7' DAY
    )
select 基准时间,
       COUNT(DISTINCT CASE WHEN 留存间隔=1 THEN id END)/COUNT(DISTINCT id) rr1,
       COUNT(DISTINCT CASE WHEN 留存间隔=3 THEN id END)/COUNT(DISTINCT id) rr3,
       COUNT(DISTINCT CASE WHEN 留存间隔=7 THEN id END)/COUNT(DISTINCT id) rr7
       ...
from 临时表 group by 基准时间
```



### 题目二

#### 表格

SQL_12 (用户登录表)

| id   | login_time          |
| ---- | ------------------- |
| 1001 | 2021-01-01 00:01:00 |
| 1002 | 2021-01-01 00:01:00 |
| 1003 | 2021-01-01 00:01:00 |
| ...  | ...                 |

#### 脚本

```sql

DROP TABLE if exists SQL_12;
CREATE TABLE SQL_12(
   user_id INT ,
   login_time DATETIME NOT NULL
);
insert into SQL_12(user_id,login_time) values (1001,'2021-01-01 00:01:00');
insert into SQL_12(user_id,login_time) values (1002,'2021-01-01 00:01:00');
insert into SQL_12(user_id,login_time) values (1003,'2021-01-01 00:01:00');
insert into SQL_12(user_id,login_time) values (1001,'2021-01-02 00:01:00');
insert into SQL_12(user_id,login_time) values (1002,'2021-01-02 00:01:00');
insert into SQL_12(user_id,login_time) values (1003,'2021-01-03 00:01:00');
insert into SQL_12(user_id,login_time) values (1004,'2021-01-03 00:01:00');
insert into SQL_12(user_id,login_time) values (1005,'2021-01-03 00:01:00');
insert into SQL_12(user_id,login_time) values (1002,'2021-01-04 00:01:00');
insert into SQL_12(user_id,login_time) values (1004,'2021-01-04 00:01:00');
insert into SQL_12(user_id,login_time) values (1005,'2021-01-04 00:01:00');
insert into SQL_12(user_id,login_time) values (1006,'2021-01-04 00:01:00');
insert into SQL_12(user_id,login_time) values (1007,'2021-01-04 00:01:00');
insert into SQL_12(user_id,login_time) values (1008,'2021-01-04 00:01:00');
insert into SQL_12(user_id,login_time) values (1001,'2021-01-05 00:01:00');
insert into SQL_12(user_id,login_time) values (1002,'2021-01-05 00:01:00');
insert into SQL_12(user_id,login_time) values (1004,'2021-01-05 00:01:00');
insert into SQL_12(user_id,login_time) values (1005,'2021-01-05 00:01:00');
insert into SQL_12(user_id,login_time) values (1006,'2021-01-05 00:01:00');
insert into SQL_12(user_id,login_time) values (1002,'2021-01-06 00:01:00');
insert into SQL_12(user_id,login_time) values (1003,'2021-01-06 00:01:00');
insert into SQL_12(user_id,login_time) values (1005,'2021-01-06 00:01:00');
insert into SQL_12(user_id,login_time) values (1006,'2021-01-06 00:01:00');
insert into SQL_12(user_id,login_time) values (1001,'2021-01-07 00:01:00');
insert into SQL_12(user_id,login_time) values (1003,'2021-01-07 00:01:00');
insert into SQL_12(user_id,login_time) values (1006,'2021-01-07 00:01:00');
insert into SQL_12(user_id,login_time) values (1007,'2021-01-07 00:01:00');

select * from SQL_12;

```

#### 问题

计算次日留存率，3日留存率（以首次登陆时间为基准

```sql
with reg as (
    select user_id, min(date(login_time)) reg_date from SQL_12 group by user_id
),
login as (
    select user_id, date(login_time) login_date from SQL_12
),
t1 as (
    SELECT a.user_id id,  a.reg_date 'reg_date', b.login_date 'login_date', 
    datediff(b.login_date, a.reg_date) 'diff'
    FROM reg a LEFT JOIN login b
    ON b.user_id = a.user_id AND b.login_date BETWEEN a.reg_date + INTERVAL '1' DAY AND a.reg_date + INTERVAL '3' DAY
)
select reg_date,
       COUNT(DISTINCT CASE WHEN diff=1 THEN id END)/COUNT(DISTINCT id) rr1,
       COUNT(DISTINCT CASE WHEN diff=3 THEN id END)/COUNT(DISTINCT id) rr3
from t1 group by reg_date;
```


