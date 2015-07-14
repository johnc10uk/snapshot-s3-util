# Updates to the CDH4 version for CDH5

Compiled against CDH 5.3.5 HBase 0.98.6 using YARN (MR2) May 2015. Likely to work with 5.3.5+ with s3a support.

Version 2.0.2-cdh5.3.5

Added a copy-from option for export. 
Default is hdfs://nameservice1/hbase or add own path e.g. for path /opt/hbase on a normal file system "-f file:///opt/hbase"

Added overwrite S3 files option for export and import snapshot

Added Bandwidth option (v0.098.3 onwards). Default 200Mb/s.

Added s3a as default (CDH 5.3.0 HADOOP-10400. Uses native Amazon aws-sdk instead of jets3t)

Tested export to s3, s3a and s3n storage.

Import will currently only work from s3n and s3a.  S3a now default, >5GB file size 
(s3n: 5GB file size limit unless use multipart file feature which allows 5TB file sizes)

Option to use fs.s3n.multipart.uploads.enabled to allow greater than 5GB HDFS files to S3n store 
(core-site.xml and jets3t.properties to increase threads from default of 2 to say 20)

s3 import fails as using s3 directory /data/default and doesn't try /.tmp or /archive as seen earlier in debug output. 
Files are in /rootpath/archive/data/default and I cannot find how to override this setting. 
Error is "java.io.IOException: No such file".

Note: Backup of a large table will require a large amount of /tmp/hadoop-hdfs/s3/ space on the datanode (>10GB). 
Ensure all datanodes /tmp have sufficient free space or sym link /tmp/hadoop-hdfs to its own disc.

# Building
```
mvn clean package
```

# Running

To create a snapshot of table test1 and immediately export to S3n:// from a NameNode server. Number of mappers should be same as number of region servers.

```
sudo -u hbase HADOOP_CLASSPATH=$(hbase classpath) hadoop jar target/snapshot-s3-util-2.0.2-cdh5.3.5.jar com.imgur.backup.SnapshotS3Util --createExport --table test1 --awsAccessKey <accessKey> --awsAccessSecret <accessSecret> --bucketName hbasebucketname --mappers 3
```
Output
```
14/11/28 14:18:01 INFO backup.SnapshotS3Util: --------------------------------------------------
14/11/28 14:18:01 INFO backup.SnapshotS3Util: Create snapshot : true
14/11/28 14:18:01 INFO backup.SnapshotS3Util: Export snapshot : true
14/11/28 14:18:01 INFO backup.SnapshotS3Util: Import snapshot : false
14/11/28 14:18:01 INFO backup.SnapshotS3Util: Table name      : test1
14/11/28 14:18:01 INFO backup.SnapshotS3Util: Snapshot name   : test1-snapshot-20141128_122849
14/11/28 14:18:01 INFO backup.SnapshotS3Util: Bucket name     : hbasebucketname
14/11/28 14:18:01 INFO backup.SnapshotS3Util: S3 Path         : /hbase/20141128
14/11/28 14:18:01 INFO backup.SnapshotS3Util: HDFS Path       : /hbase
14/11/28 14:18:01 INFO backup.SnapshotS3Util: Snapshot From Url : hdfs://nameservice1/hbase
14/11/28 14:18:01 INFO backup.SnapshotS3Util: Mappers         : 3
14/11/28 14:18:01 INFO backup.SnapshotS3Util: Bandwidth       : 200
14/11/28 14:18:01 INFO backup.SnapshotS3Util: S3 protocol     : s3a://
14/11/28 14:18:01 INFO backup.SnapshotS3Util: HBase Snapshot TTL    : 0
14/11/28 14:18:01 INFO backup.SnapshotS3Util: Overwrite S3 files  : false
14/11/28 14:18:01 INFO backup.SnapshotS3Util: --------------------------------------------------
14/11/28 14:18:01 INFO backup.SnapshotS3Util: Creating snapshot...
14/11/28 14:18:01 INFO snapshot.ExportSnapshot: Export Completed: test1-snapshot-20141128_122849
14/11/28 14:18:01 INFO backup.SnapshotS3Util: Successfully exported snapshot 'test1-snapshot-20141128_122849' to S3
14/11/28 14:18:01 INFO backup.SnapshotS3Util: Complete

```
Import from s3a:// table backup in to Hbase

```
hbase shell
delete_snapshot 'test1-snapshot-20140828_154448'
scan 'test1'
exit

Ensure hbase user can write to HDFS /user

sudo -u hdfs hadoop fs -chmod 777 /user

Import snapshot from s3n store directory /hbase (-p) to HDFS /hbase (-d)

sudo -u hbase HADOOP_CLASSPATH=$(hbase classpath) hadoop jar target/snapshot-s3-util-2.0.2-cdh5.3.5.jar com.imgur.backup.SnapshotS3Util --import --snapshot test1-snapshot-20140828_154448 -d /hbase -p /hbase --awsAccessKey <accessKey> --awsAccessSecret <accessSecret> --bucketName <bucketName> --mappers <numMaps>

Confirm table can be restored

hbase shell
disable 'test1'
drop 'test1'
restore_snapshot 'test1-snapshot-20140822_101514'
scan 'test1' 
```

Output
```
14/08/28 15:48:58 INFO backup.SnapshotS3Util: --------------------------------------------------
14/08/28 15:48:58 INFO backup.SnapshotS3Util: Create snapshot : false
14/08/28 15:48:58 INFO backup.SnapshotS3Util: Export snapshot : false
14/08/28 15:48:58 INFO backup.SnapshotS3Util: Import snapshot : true
14/08/28 15:48:58 INFO backup.SnapshotS3Util: Table name      : null
14/08/28 15:48:58 INFO backup.SnapshotS3Util: Snapshot name   : test1-snapshot-20140828_154448
14/08/28 15:48:58 INFO backup.SnapshotS3Util: Bucket name     : hbasebucketname
14/08/28 15:48:58 INFO backup.SnapshotS3Util: S3 Path         : /hbase
14/08/28 15:48:58 INFO backup.SnapshotS3Util: HDFS Path       : /hbase
14/08/28 15:48:58 INFO backup.SnapshotS3Util: Snapshot From Url : hdfs://nameservice1/hbase
14/08/28 15:48:58 INFO backup.SnapshotS3Util: Mappers         : 3
14/08/28 15:48:58 INFO backup.SnapshotS3Util: Bandwidth       : 200
14/08/28 15:48:58 INFO backup.SnapshotS3Util: s3 protocol     : s3a://
14/08/28 15:48:58 INFO backup.SnapshotS3Util: Snapshot TTL    : 0
14/08/28 15:48:58 INFO backup.SnapshotS3Util: Overwrite S3 files  : false
14/08/28 15:48:58 INFO backup.SnapshotS3Util: --------------------------------------------------
14/08/28 15:49:24 INFO snapshot.ExportSnapshot: Finalize the Snapshot Export
14/08/28 15:49:24 INFO snapshot.ExportSnapshot: Verify snapshot integrity
14/08/28 15:49:24 INFO Configuration.deprecation: fs.default.name is deprecated. Instead, use fs.defaultFS
14/08/28 15:49:24 INFO snapshot.ExportSnapshot: Export Completed: test1-snapshot-20140828_154448
14/08/28 15:49:24 INFO backup.SnapshotS3Util: Successfully imported snapshot 'test1-snapshot-20140828_154448' from S3
14/08/28 15:49:24 INFO backup.SnapshotS3Util: Complete
```
# Options
```
usage: SnapshotS3Util [-a] -b <arg> -c | -e | -f <arg> | -i | -x [-d <arg>] -k <arg>
       [-l <arg>] [-m <arg>] [-n <arg>] [-p <arg>] -s <arg> [-t <arg>] [-w <arg>] [-r <arg> ]
       
Backup utility for creating snapshots and exporting/importing to and from S3

 -a,--s3                      Use s3 protocol instead of s3a. Currently not
                              working for s3 import (file import path issue)                           
 -b,--bucketName <arg>        The S3 bucket name where snapshots are
                              stored
 -c,--create                  Create HBase snapshot
 -d,--hdfsPath <arg>          The snapshot directory in HDFS. Default is
                              '/hbase'
 -e,--export                  Export HBase snapshot to S3
 -f,--copy-from <arg>         Export from snapshot URL. Default is 
                              hdfs://nameservice1/hbase
 -i,--import                  Import HBase snapshot from S3. May need to
                              run as hbase user if importing into HBase
 -k,--awsAccessKey <arg>      The AWS access key
 -l,--snapshotTtl <arg>       Delete snapshots older than this value
                              (seconds) from running HBase cluster
 -m,--mappers <arg>           The number of parallel copiers if copying (same as number of region server)
                              to/from S3. Default: 1
 -n,--snapshot <arg>          The snapshot name. Required for importing
                              from S3
 -p,--s3Path <arg>            The snapshot directory in S3. Default is
                              '/hbase'
 -r,--bandwidth <arg>         The network bandwidth copying to/from S3 in
                              Mb/s (v0.098.3 onwards). Default: 200
 -s,--awsAccessSecret <arg>   The AWS access secret string
 -t,--table <arg>             The table name to create a snapshot from.
                              Required for creating a snapshot
 -x,--createExport            Create HBase snapshot AND export to S3
 -w,--overwrite <arg>         Overwrite S3 files if already exist. Default is false
 -y,--s3n                     Use s3n protocol instead of s3a.

```

##Required HDFS core-site.xml 

```
<!-- Amazon S3 -->
<property>
  <name>fs.s3a.impl</name>
  <value>org.apache.hadoop.fs.s3a.S3AFileSystem</value>
</property>
<property>
  <name>fs.AbstractFileSystem.s3a.impl</name>
  <value>org.apache.hadoop.fs.s3a.S3AFileSystem</value>
</property>
```

