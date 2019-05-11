# idbroker-demo
datalake, role maping, personas and activity

## DataLake Bucket Structure

```CDP-DL-1
/
|-INPUT
	|- tweets
		|- summary
			|- 2019
				|- 05
					|- 01...
					|- 10
						389457483kkjkljk
|- OUTPUT
	|- shared
		|- tweets
			|- summary
				|- 2019
					|- 05
						|- 01...
						|- 10
							389457483kkjkljk
		|- sentiments
			|- data
			|- logs
		|- financial-trends
			|- data
			|- logs
		|- insurance
			|- FL
		|- teams
			|- team-A
				|- data
	|- hive
		|- tables
		|- logs
	|- spark ## needs access to hive data
		|- ???
|- BACKUPS```

## Roles to Protect DataLake Structure

INGEST ROLE - read-write INPUT only
SCIENTEST ROLE - read only INPUT - read-write OUTPUT/shared/sentiments&&financial-trends
TEAM-A SCIENTEST ROLE - read only INPUT - read-write OUTPUT/shared/sentiments&&financial-trends+teams/team-A
TABULAR-APP-ROLE (hive) - read-write OUTPUT/shared/** + read-write OUTPUT/hive/**
BACKUP-SOURCE-ROLE
BACKUP-DEST-ROLE

// HDF ingest of twitter data
nifi-ingest user: 
Nifi-Ingest
AKIA6IVID6ZVILWZIDVC
cCCqjOzSEMelSJTECCd9LBJJHokBvHmDs36m0naM

given the LJM-CDP-INGEST-ROLE

// shell based
nifi:cloudera with mapping of ingest group -> LJM-CDP-INGEST-ROLE

lmccay:cloudera with mapping of lmccay user -> LJM-CDP-DATA-SCIENTEST

systest:cloudera with mapping of systest user -> LJM-CDP-QE-DATA-SCIENTEST

bdr:cloudera with mapping of backup-source group -> LJM-CDP-BACKUP-SOURCE-ROLE, LJM-CDP-BACKUP-DEST-ROLE

Usecases:

1. No access without kinit as a mapped identity
2. Ingest user - Read/Write access to INPUT paths of data lake bucket - no access to OUTPUT paths
    - external user with INGESTER Role
    - cluster user that maps to INGESTER Role
3. Data Scientest user - Read access to INPUT paths and Read/Write access to OUTPUT paths
    - copy tweets to OUTPUT
    - fail to write to INPUT
4. Backup-source user - Read only access to INPUT and OUTPUT
    - distcp specifying specific group mapping to use for backup-source to gs:// bucket
5. Backup-dest user - write only access to backup location
    - distcp specifying specific group mapping to use for backup-dest from gs:// bucket to s3a:// backup location
6. QE DataScientest - a QE role that has Read access to INPUT and Read/Write to OUTPUT
7. Specific service (hive) user - Read/Write access to service specific OUTPUT paths
    - service user mapping to a Role with Read of OUTPUT and Read/Write to Hive
    - show policy file
8. Hive use of IDBroker
    - SQL queries against florida insurance data
    - SQL queries against HDF flown tweets
9. Spark use of IDBroker
    - simple word count of insurance data

# Personas - separate tabs in terminal

1. systest - Data scientist - ETL (as systest)
2. BDR - backup source - readonly (as bdr)
3. lmccay - Data scientist - hive user (as hive)
3. sparky - Data scientist - spark user (as sparky)
4. nifi - Ingest - ingest user read/write to INPUT only (as nifi)

## ETL Data Worker (distcp)
hadoop distcp s3a://cldr-cdp-dl-1/INPUT/tweets/summary/2019/05/10/2019-05-10T19-23-32-100-summary-tweets-168746aa-2620-4a76-9b96-10612e4799f2.orc s3a://cldr-cdp-dl-1/OUTPUT/shared/tweets.orc

## spark-shell data scientest

val sc.hadoopConfiguration.set("fs.s3a.delegation.token.binding", "org.apache.knox.gateway.cloud.idbroker.s3a.IDBDelegationTokenBinding")

val sc.hadoopConfiguration.set("fs.s3a.ext.cab.address", "https://karthik-sdx-base-1.vpc.cloudera.com:8444/gateway")

val textFile = spark.read.textFile("s3a://cldr-cdp-dl-1/OUTPUT/shared/insurance/FL/FL_insurance_sample.csv")

textFile.count()

scala> val orcfile = "s3a://cldr-cdp-dl-1/INPUT/tweets/summary/2019/05/10/2019-05-10T19-23-32-100-summary-tweets-168746aa-2620-4a76-9b96-10612e4799f2.orc"
orcfile: String = s3a://cldr-cdp-dl-1/INPUT/tweets/summary/2019/05/10/2019-05-10T19-23-32-100-summary-tweets-168746aa-2620-4a76-9b96-10612e4799f2.orc

scala> val sqlContext = new org.apache.spark.sql.SQLContext(sc)
warning: there was one deprecation warning; re-run with -deprecation for details
sqlContext: org.apache.spark.sql.SQLContext = org.apache.spark.sql.SQLContext@4a467f08

scala> val df = sqlContext.read.format("orc").load(orcfile)
19/05/11 09:17:59 WARN impl.MetricsConfig: Cannot locate configuration: tried hadoop-metrics2-s3a-file-system.properties,hadoop-metrics2.properties
df: org.apache.spark.sql.DataFrame = [id: bigint, id_str: string ... 4 more fields]

scala> df.show()
+-------------------+-------------------+--------------------+-------------+----+--------------------+
|                 id|             id_str|          created_at| timestamp_ms|lang|                text|
+-------------------+-------------------+--------------------+-------------+----+--------------------+
|1126930588533952512|1126930588533952512|Fri May 10 19:22:...|1557516164321|  en|RT @NSA_QIL: http...|
|1126930588257198081|1126930588257198081|Fri May 10 19:22:...|1557516164255|  en|RT @MauliConty: C...|
|1126930593751687169|1126930593751687169|Fri May 10 19:22:...|1557516165565|  en|RT @DebbieDebbieD...|
|1126930597111160832|1126930597111160832|Fri May 10 19:22:...|1557516166366|  en|[Update: RAW in N...|
|1126930589540593664|1126930589540593664|Fri May 10 19:22:...|1557516164561|  en|@bottomrung @Mich...|
|1126930599145611264|1126930599145611264|Fri May 10 19:22:...|1557516166851|  en|RT @fs0c131y: I t...|
|1126930603151122432|1126930603151122432|Fri May 10 19:22:...|1557516167806|  en|@SeanQuigley87 @P...|
|1126930604010942465|1126930604010942465|Fri May 10 19:22:...|1557516168011|  en|Newcastle-upon-Ty...|
|1126930601662210050|1126930601662210050|Fri May 10 19:22:...|1557516167451|  en|RT @jaffathecake:...|
|1126930612827279360|1126930612827279360|Fri May 10 19:22:...|1557516170113|  en|WHOA! This is hig...|
|1126930614341525504|1126930614341525504|Fri May 10 19:22:...|1557516170474|  en|@photokool6 Hi Sa...|
|1126930616740667393|1126930616740667393|Fri May 10 19:22:...|1557516171046|  en|Catch up on the b...|
|1126930615855611906|1126930615855611906|Fri May 10 19:22:...|1557516170835|  en|RT @coughetycough...|
|1126930615486550016|1126930615486550016|Fri May 10 19:22:...|1557516170747|  en|Gomorrah S.4 comi...|
|1126930614932914176|1126930614932914176|Fri May 10 19:22:...|1557516170615|  en|I see some of my ...|
|1126930617743093760|1126930617743093760|Fri May 10 19:22:...|1557516171285|  en|          Beautiful.|
|1126930622176530433|1126930622176530433|Fri May 10 19:22:...|1557516172342|  en|RT @TrulyTafakari...|
|1126930622797230080|1126930622797230080|Fri May 10 19:22:...|1557516172490|  en|RT @matthewstolle...|
|1126930623275393024|1126930623275393024|Fri May 10 19:22:...|1557516172604|  en|RT @22shtnamas: H...|
|1126930623661248513|1126930623661248513|Fri May 10 19:22:...|1557516172696|  en|RT @angryislander...|
+-------------------+-------------------+--------------------+-------------+----+--------------------+
only showing top 20 rows


scala> df.printSchema()
root
 |-- id: long (nullable = true)
 |-- id_str: string (nullable = true)
 |-- created_at: string (nullable = true)
 |-- timestamp_ms: string (nullable = true)
 |-- lang: string (nullable = true)
 |-- text: string (nullable = true)


scala> df.select("text").show()
+--------------------+                                                          
|                text|
+--------------------+
|RT @NSA_QIL: http...|
|RT @MauliConty: C...|
|RT @DebbieDebbieD...|
|[Update: RAW in N...|
|@bottomrung @Mich...|
|RT @fs0c131y: I t...|
|@SeanQuigley87 @P...|
|Newcastle-upon-Ty...|
|RT @jaffathecake:...|
|WHOA! This is hig...|
|@photokool6 Hi Sa...|
|Catch up on the b...|
|RT @coughetycough...|
|Gomorrah S.4 comi...|
|I see some of my ...|
|          Beautiful.|
|RT @TrulyTafakari...|
|RT @matthewstolle...|
|RT @22shtnamas: H...|
|RT @angryislander...|
+--------------------+
only showing top 20 rows


## Hive External Table csv

[root@karthik-sdx-base-1 ~]# beeline
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/opt/cloudera/parcels/CDH-6.0.99-1.cdh6.0.99.p0.129/jars/log4j-slf4j-impl-2.10.0.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/opt/cloudera/parcels/CDH-6.0.99-1.cdh6.0.99.p0.129/jars/slf4j-log4j12-1.6.1.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.apache.logging.slf4j.Log4jLoggerFactory]
WARNING: Use "yarn jar" to launch YARN applications.
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/opt/cloudera/parcels/CDH-6.0.99-1.cdh6.0.99.p0.129/jars/log4j-slf4j-impl-2.10.0.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/opt/cloudera/parcels/CDH-6.0.99-1.cdh6.0.99.p0.129/jars/slf4j-log4j12-1.6.1.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.apache.logging.slf4j.Log4jLoggerFactory]
Beeline version 3.1.0.6.0.99.0-129 by Apache Hive


beeline> !connect jdbc:hive2://karthik-sdx-base-1.vpc.cloudera.com:10000/default;principal=hive/_HOST@VPC.CLOUDERA.COM
Connecting to jdbc:hive2://karthik-sdx-base-1.vpc.cloudera.com:10000/default;principal=hive/_HOST@VPC.CLOUDERA.COM
Connected to: Apache Hive (version 3.1.0.6.0.99.0-129)
Driver: Hive JDBC (version 3.1.0.6.0.99.0-129)
Transaction isolation: TRANSACTION_REPEATABLE_READ

### Create external table

0: jdbc:hive2://karthik-sdx-base-1.vpc.cloude> CREATE EXTERNAL TABLE FL_INS(`policyID` STRING,`statecode` STRING,`county` STRING,`eq_site_limit` STRING,`hu_site_limit` STRING,`fl_site_limit` STRING,`fr_site_limit` STRING,`tiv_2011` STRING,`tiv_2012` STRING,`eq_site_deductible` STRING,`hu_site_deductible` STRING,`fl_site_deductible` STRING,`fr_site_deductible` STRING,`point_latitude` STRING,`point_longitude` STRING,`line` STRING,`construction` STRING,`point_granularity` STRING) ROW FORMAT DELIMITED FIELDS TERMINATED BY "," LOCATION "s3a://cldr-cdp-dl-1/OUTPUT/shared/insurance/FL/" TBLPROPERTIES ("skip.header.line.count"="1");

### Execute query

0: jdbc:hive2://karthik-sdx-base-1.vpc.cloude> select policyid, fl_site_deductible, line from fl_ins LIMIT 10;
INFO  : Compiling command(queryId=hive_20190511114553_7e44a7d9-9044-432a-b9b3-94954004e457): select policyid, fl_site_deductible, line from fl_ins LIMIT 10
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Returning Hive schema: Schema(fieldSchemas:[FieldSchema(name:policyid, type:string, comment:null), FieldSchema(name:fl_site_deductible, type:string, comment:null), FieldSchema(name:line, type:string, comment:null)], properties:null)
INFO  : Completed compiling command(queryId=hive_20190511114553_7e44a7d9-9044-432a-b9b3-94954004e457); Time taken: 1.438 seconds
INFO  : Executing command(queryId=hive_20190511114553_7e44a7d9-9044-432a-b9b3-94954004e457): select policyid, fl_site_deductible, line from fl_ins LIMIT 10
INFO  : Completed executing command(queryId=hive_20190511114553_7e44a7d9-9044-432a-b9b3-94954004e457); Time taken: 0.763 seconds
INFO  : OK
+-----------+---------------------+--------------+
| policyid  | fl_site_deductible  |     line     |
+-----------+---------------------+--------------+
| 119736    | 0                   | Residential  |
| 448094    | 0                   | Residential  |
| 206893    | 0                   | Residential  |
| 333743    | 0                   | Residential  |
| 172534    | 0                   | Residential  |
| 785275    | 0                   | Residential  |
| 995932    | 0                   | Commercial   |
| 223488    | 0                   | Residential  |
| 433512    | 0                   | Residential  |
| 142071    | 0                   | Residential  |
+-----------+---------------------+--------------+
10 rows selected (8.23 seconds)


## Hive and ORC

### create external table

set hive.cli.errors.ignore=true;
set hive.exec.dynamic.partition.mode=nonstrict;
set hive.exec.dynamic.partition=true;
set hive.enforce.bucketing = true;
SET hive.exec.compress.output=true;
SET avro.output.codec=snappy;

CREATE DATABASE IF NOT EXISTS twitter;
USE twitter;

CREATE EXTERNAL TABLE IF NOT EXISTS tweetsummary_part 
(id BIGINT, id_str STRING, created_at STRING, 
timestamp_ms STRING, lang STRING, text STRING) 
PARTITIONED BY (year string, month string, day string)
STORED AS ORC
location 's3a://cldr-cdp-dl-1/INPUT/tweets/summary/';

ALTER TABLE tweetsummary_part ADD PARTITION (year='2019',month='05',day='10') 
location 's3a://cldr-cdp-dl-1/INPUT/tweets/summary/2019/05/10';

### query twitter table

0: jdbc:hive2://karthik-sdx-base-1.vpc.cloude> select id, text from tweetsummary_part LIMIT 10;
INFO  : Compiling command(queryId=hive_20190511121745_378db5aa-9b04-4374-9cd5-634525e70e79): select id, text from tweetsummary_part LIMIT 10
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Returning Hive schema: Schema(fieldSchemas:[FieldSchema(name:id, type:bigint, comment:null), FieldSchema(name:text, type:string, comment:null)], properties:null)
INFO  : Completed compiling command(queryId=hive_20190511121745_378db5aa-9b04-4374-9cd5-634525e70e79); Time taken: 0.142 seconds
INFO  : Executing command(queryId=hive_20190511121745_378db5aa-9b04-4374-9cd5-634525e70e79): select id, text from tweetsummary_part LIMIT 10
INFO  : Completed executing command(queryId=hive_20190511121745_378db5aa-9b04-4374-9cd5-634525e70e79); Time taken: 0.013 seconds
INFO  : OK
+----------------------+----------------------------------------------------+
|          id          |                        text                        |
+----------------------+----------------------------------------------------+
| 1126928634839732224  | RT @the_JACKTRADO: I asked Google assistant what women want.

I had to shut down my phone cause it won't stop talking. |
| 1126928633220743169  | RT @NHSBT_RD: Today's Google Doodle Honors the life and work of haematologist Dr Lucy Wills, on what would have been her 131st Birthday. He… |
| 1126928636811055105  | .. retarded delusion .. https://t.co/JKcK3H52Kg via @YouTube .. great job and well done and the white psychotic cra… https://t.co/GNUbo5PhZI |
| 1126928641005371392  | RT @MilesTheIcon: Google when you sign in from a new device https://t.co/PBk0mAh0yB |
| 1126928642850869248  | Hey @Shopify are you seriously going to kill off your Google Shopping app and replace it with this new one that has… https://t.co/KVhMb9xAQl |
| 1126928645975678976  | RT @RealCandaceO: WOW!!! The students of the Colorado shooting WALKED OUT of the vigil saying that they would NOT allow their pain to be po… |
| 1126928648232153091  | Google is helping train military spouses for remote IT work https://t.co/EzQUENyLOh |
| 1126928650497134598  | RT @RealSUPREME_G: IT'S OFFICIAL

My New Single "Brand New Drip" Is Now Available On Spotify, Tidal, Amazon And Google Play And It's Coming… |
| 1126928650425831425  | RT @HMLoeschMcK: @AOC perhaps you should rethink your comments on the #HeartBeatLaws it is enough time! I also recommend all you abortion l… |
| 1126928654024495104  | @matthallsy Google support and tell me the definition. I do t understand when football changed and all these spoilt… https://t.co/lAzoNw0JRh |
+----------------------+----------------------------------------------------+
10 rows selected (2.054 seconds)
0: jdbc:hive2://karthik-sdx-base-1.vpc.cloude> Closing: 0: jdbc:hive2://karthik-sdx-base-1.vpc.cloudera.com:10000/default;principal=hive/_HOST@VPC.CLOUDERA.COM
