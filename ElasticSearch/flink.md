https://github.com/alibaba/canal

### mysql
对于自建 MySQL , 需要先开启 Binlog 写入功能，配置 binlog-format 为 ROW 模式，my.cnf 中配置如下
```
[mysqld]
log-bin=mysql-bin # 开启 binlog
binlog-format=ROW # 选择 ROW 模式
server_id=1 # 配置 MySQL replaction 需要定义，不要和 canal 的 slaveId 重复

#不同步哪些数据库
binlog-ignore-db = mysql
binlog-ignore-db = test
binlog-ignore-db = information_schema
```
在MySQL中执行`show binlog events;`  和 `show global variables like "binlog%"` 查看配置是否生效


### canal server
这里，mysql，cannal，es 都使用了同一个网络 elastaisearch_default

docker pull canal/canal-server:v1.1.1

docker run --name canal111 \
--network elastaisearch_default \
-e canal.instance.master.address=mysql8:3306 \
-e canal.instance.dbUsername=root\
-e canal.instance.dbPassword=123456\
-e canal.instance.connectionCharset=UTF-8 \
-e canal.instance.gtidon=false \
-e canal.instance.filter.regex=.*\\\\\\..*  \
-p 11111:11111 \
-d canal/canal-server:v1.1.1

docker run --name canal  --network elastaisearch_default -e canal.instance.master.address=mysql8:3306 -e canal.instance.dbUsername=root -e canal.instance.dbPassword=123456  -e canal.instance.connectionCharset=UTF-8 -e canal.instance.gtidon=false -e canal.instance.filter.regex=.*\\..*  -p 11111:11111 -d canal/canal-server:v1.1.5


docker run --name canal  --network elastaisearch_default -e canal.instance.master.address=mysql8:3306 -e canal.instance.dbUsername=root -e canal.instance.dbPassword=123456  -e canal.instance.connectionCharset=UTF-8 -e canal.instance.filter.regex=.*\\..*  -p 11111:11111 -d canal/canal-server:v1.1.6




flink 有丰富的connector生态，不管是上有还是下游都有丰富

1，下载flink 1.16.0 docke compose
2，安装MySQL、es
3.启动flink sql的sql-client

https://repo1.maven.org/maven2/com/ververica/flink-sql-connector-postgres-cdc/2.3.0/

https://repo.maven.apache.org/maven2/org/apache/flink/flink-sql-connector-elasticsearch7/3.0.0-1.16/flink-sql-connector-elasticsearch7-3.0.0-1.16.jar;

### 修改mysql 配置文件，开启binlog
[mysqld]
log-bin=mysql-bin # 开启 binlog
binlog-format=ROW # 选择 ROW 模式
server_id=1

两边是数据库必须设置相同的时区
show variables like '%time_zone%';
set global time_zone='+8:00';
set time_zone='+8:00';


SET 'table.local-time-zone' = 'Asia/Shanghai';






SET execution.checkpointing.interval = 3s;
CREATE TABLE orders (
   order_id INT,
   order_date TIMESTAMP(3),
   customer_name STRING,
   price DECIMAL(10, 5),
   product_id INT,
   product_name STRING,
   PRIMARY KEY (order_id) NOT ENFORCED
 ) WITH (
   'connector' = 'mysql-cdc',
   'hostname' = '106.53.5.146',
   'port' = '3306',
   'username' = 'mysql57',
   'password' = 'shijinting0510',
   'database-name' = 'mydb',
   'table-name' = 'orders'
 );

 CREATE TABLE top_products(
 product_name STRING,
 sales_cnt BIGINT NOT NULL,
 PRIMARY KEY (product_name) NOT ENFORCED
 ) WITH (
 'connector' = 'elasticsearch-7',
 'hosts' = 'http://192.168.48.5:9200',
 'index' = 'top_products'
 );
 
INSERT INTO top_products  
select product_name, count(*) as cnt from orders group by product_name;