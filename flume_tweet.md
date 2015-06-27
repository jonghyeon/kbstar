#flume 실습2
##사전 준비

##Step 1: Download and Extract Data File


##Step 2: data 수집
###### 설정파일
```
TwitterAgent.sources = Twitter
TwitterAgent.channels = MemChannel
#TwitterAgent.sinks = HDFS
TwitterAgent.sinks = file_roll

TwitterAgent.sources.Twitter.type = com.sstrato.flume.source.TwitterSource
TwitterAgent.sources.Twitter.channels = MemChannel
TwitterAgent.sources.Twitter.consumerKey = wveu7QC2pwqvLW7bc4yuVlrXu
TwitterAgent.sources.Twitter.consumerSecret = UtxuKqb2805Sc8ZRklly2XMbwjG1BuFOjYfygfFEIdkkCmIJta
TwitterAgent.sources.Twitter.accessToken = 1709179110-awFIGgl1zqLTleYf928srEn6dEgjUpKf42WjX2u
TwitterAgent.sources.Twitter.accessTokenSecret = mB5xJKDEGrRA8UxCbbeaL3obSW6MBrCWFXbjQ1PGMKtvE
#TwitterAgent.sources.Twitter.keyword = hadoop, big data, analytics, bigdata, cloudera, data science, data scientiest, business intelligence, mapreduce, data warehouse, data warehousing, mahout, hbase, nosql, newsql, businessintelligence, cloudcomputing
TwitterAgent.sources.Twitter.keyword = 복면가왕
#TwitterAgent.sinks.HDFS.channel = MemChannel
#TwitterAgent.sinks.HDFS.type = hdfs
#TwitterAgent.sinks.HDFS.hdfs.path = hdfs://hadoop1:8020/user/flume/tweets/%Y/%m/%d/%H/
#TwitterAgent.sinks.HDFS.hdfs.fileType = DataStream
#TwitterAgent.sinks.HDFS.hdfs.writeFormat = Text
#TwitterAgent.sinks.HDFS.hdfs.batchSize = 1000
#TwitterAgent.sinks.HDFS.hdfs.rollSize = 0
#TwitterAgent.sinks.HDFS.hdfs.rollCount = 10000

TwitterAgent.sinks.file_roll.type = file_roll
TwitterAgent.sinks.file_roll.channel = MemChannel
TwitterAgent.sinks.file_roll.sink.directory = /home/hadoop/flume/logs
TwitterAgent.channels.MemChannel.type = memory
TwitterAgent.channels.MemChannel.capacity = 10000
TwitterAgent.channels.MemChannel.transactionCapacity = 100
```
###### 수집
````
[hadoop@hadoop01 flume]$ ./bin/flume-ng agent -n TwitterAgent -f ./conf/twitter.conf
````


##Step 3-1: hive table 만들기 - 실시간 수집

```
0: jdbc:hive2://hadoop01:10000> ADD JAR /home/hadoop/hive/udf/hive-serdes-1.0-SNAPSHOT.jar;
```
```
0: jdbc:hive2://hadoop01:10000> CREATE EXTERNAL TABLE tweets (
id BIGINT, created_at STRING,
source STRING, favorited BOOLEAN,
retweeted_status STRUCT<
text:STRING,
user:STRUCT<screen_name:STRING,name:STRING>,
retweet_count:INT>,
entities STRUCT<
urls:ARRAY<STRUCT<expanded_url:STRING>>,
user_mentions: ARRAY<STRUCT<screen_name:STRING,name:STRING>>,
hashtags:ARRAY<STRUCT<text:STRING>>>,
text STRING,
user STRUCT<
screen_name:STRING,
name:STRING,
friends_count:INT,
followers_count:INT,
statuses_count:INT,
verified:BOOLEAN,
utc_offset:INT,
time_zone:STRING>,
in_reply_to_screen_name STRING
)
PARTITIONED BY (datehour INT)
ROW FORMAT SERDE 'com.sstrato.hive.serde.JSONSerDe'
LOCATION '/user/flume/tweets';
```
```
0: jdbc:hive2://hadoop01:10000> ALTER TABLE tweets ADD IF NOT
EXISTS PARTITION (datehour =
2014041301) LOCATION '/user/flume/tweets/
2014/04/13/01';
```
##Step 3-2: hive table 만들기 - 미리 수집된 데이터
```
0: jdbc:hive2://hadoop01:10000> ADD JAR /home/hadoop/hive/udf/hive-serdes-1.0-SNAPSHOT.jar;
0: jdbc:hive2://hadoop01:10000> CREATE EXTERNAL TABLE tweets (
id BIGINT, created_at STRING,
source STRING, favorited BOOLEAN,
retweeted_status STRUCT<
text:STRING,
user:STRUCT<screen_name:STRING,name:STRING>,
retweet_count:INT>,
entities STRUCT<
urls:ARRAY<STRUCT<expanded_url:STRING>>,
user_mentions: ARRAY<STRUCT<screen_name:STRING,name:STRING>>,
hashtags:ARRAY<STRUCT<text:STRING>>>,
text STRING,
user STRUCT<
screen_name:STRING,
name:STRING,
friends_count:INT,
followers_count:INT,
statuses_count:INT,
verified:BOOLEAN,
utc_offset:INT,
time_zone:STRING>,
in_reply_to_screen_name STRING
)
PARTITIONED BY (datehour INT)
ROW FORMAT SERDE 'com.sstrato.hive.serde.JSONSerDe';
```
```
0: jdbc:hive2://hadoop01:10000> load data local inpath '/home/hadoop/flume/logs/*' overwrite into table tweets partition (datehour=20150621);
```

##Step 4: 분석
###### 수집 대상중 가장 많은 time zone
```
0: jdbc:hive2://hadoop01:10000> SELECT
user.time_zone,
SUBSTR(created_at, 0, 3),
COUNT(*) AS total_count
FROM tweets
WHERE user.time_zone IS NOT NULL
GROUP BY
user.time_zone,
SUBSTR(created_at, 0, 3)
ORDER BY total_count DESC
LIMIT 15;
```
###### 수집 대상 중 가장 많은 hashtag
````
0: jdbc:hive2://hadoop01:10000> SELECT
LOWER(hashtags.text),
COUNT(*) AS total_count
FROM tweets
LATERAL VIEW EXPLODE(entities.hashtags)
t1 AS hashtags
GROUP BY LOWER(hashtags.text)
ORDER BY total_count DESC
LIMIT 15;
````
