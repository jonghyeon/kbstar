#sqoop 실습
##사전 준비

###### hadoop 실행
```
$HADOOP_HOME/sbin/start-all.sh
```
###### hive 실행
```
$HIVE_HOME/bin/hive --service metastore > /dev/null 2>&1 &
$HIVE_HOME/bin/hive --service hiveserver2 > /dev/null 2>&1 &
```
###### mysql tables 생성
```
$ cd /sql (sql 파일(mysqlsampledatabase.sql)이 있는 디렉토리로 이동)
$ mysql -u 접속계정 -p (mysql 접속)
mysql> source ./mysqlsampledatabase.sql
mysql> CREATE DATABASE test
mysql> CREATE TABLE FIREWALL_LOGS(
time varchar(50),
ip varchar(50),
country varchar(50),
status varchar(50));
```

##Step 1: Download and Extract Data File
```
[hadoop@hadoop01 ~]$ wget http://apache.mirror.cdnetworks.com/sqoop/1.4.6/sqoop-1.4.6.bin__hadoop-2.0.4-alpha.tar.gz
```

##Step 3: RDBMS table을 hdfs로 가져오기
```
[hadoop@hadoop01 ~]$ ./sqoop/bin/sqoop import --connect jdbc:mysql://hadoop01/classicmodels --username hive --password hive --table customers -m 1 --hive-import
```
```
0: jdbc:hive2://hadoop01:10000> show tables;
+----------------+--+
|    tab_name    |
+----------------+--+
| customers      |
| firewall_logs  |
| stocks         |
+----------------+--+
3 rows selected (0.038 seconds)
0: jdbc:hive2://hadoop01:10000> select * from customers limit 10;

```
###### DB내 전체 테이블 Import
```
sqoop import-all-tables \
   --connect jdbc:mysql://hadoop01/classicmodels \
   --username hive \
   --password hive \
   --exclude-tables cities,countries \
   --warehouse-dir /user/hive/warehouse/input/
```
###### 일부 데이터, 조건에 맞는 데이터 가져오기
```
   A) --incremental 옵션 사용
      acclog 테이블의 seq 칼럼값이 8910000 이상인 것만 import
        sqoop import \
        --connect jdbc:mysql://mysql.example.com/mydb \
        --username sqoop \
        --password sqoop \
        --table acclog \
        --target-dir /teststore/input/acclog \
        --num-mappers 10 \
        --incremental append \
        --check-column seq \
        --last-value 8910000
   B) 특정 Query 조건에 맞는 데이터 가져오기
        sqoop import \
        --connect jdbc:mysql://mysql.example.com/mydb \
        --username sqoop \
        --password sqoop \
        --target-dir /teststore/input/acclog \
        --query 'SELECT seq, inputdate, searchkey \
                   FROM acclog
                   WHERE seq > 10' \
        --num-mappers 10
```

##Step 3: hadoop 에서 RDBMS로 가져오기
```
[hadoop@hadoop01 ~]$ ./sqoop/bin/sqoop export --connect jdbc:mysql://hadoop01/test --table FIREWALL_LOGS --username hive --password hive --export-dir=/eventlog -m 1 --input-fields-terminated-by '|'
```