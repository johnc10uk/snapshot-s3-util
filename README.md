# Updates to the CDH4 version

Compiled against CDH 5.1.3 using YARN (MR2) September 2014
Version 2.0.1
Added a copy-from option for export
Test export to s3 and s3n storage
Import will currently only work from s3n, now default (5GB file size limit unless use multipart file feature which allows 5TB file sizes)
Option to use fs.s3n.multipart.uploads.enabled to allow greater than 5GB HDFS files to S3n store (core-site.xml and jets3t.properties to increase threads from default of 2 to say 20)
s3 import fails as using s3 directory /data/default and doesn't try /.tmp or /archive as seen earlier in debug output. Files are in /rootpath/archive/data/default and I cannot find how to override this setting. Error is java.io.IOException: No such file.

# Building
```
mvn clean package
```

# Running

To create a snapshot of table test1 and immediately export to S3n:// from a NameNode server

Note: hbase user shell may be set to /bin/false, set to /bin/bash.

```
grep /etc/passwd hbase
 hbase:x:985:984:HBase:/var/run/hbase:/bin/bash
sudo -u hbase HADOOP_CLASSPATH=$(hbase classpath) hadoop jar target/snapshot-s3-util-2.0.1.jar com.imgur.backup.SnapshotS3Util --createExport --table test5 --awsAccessKey <accessKey> --awsAccessSecret <accessSecret> --bucketName <bucketName> --mappers <numMaps>
```
Output
```
14/08/28 15:44:48 INFO backup.SnapshotS3Util: --------------------------------------------------
14/08/28 15:44:48 INFO backup.SnapshotS3Util: Create snapshot : true
14/08/28 15:44:48 INFO backup.SnapshotS3Util: Export snapshot : true
14/08/28 15:44:48 INFO backup.SnapshotS3Util: Import snapshot : false
14/08/28 15:44:48 INFO backup.SnapshotS3Util: Table name      : test1
14/08/28 15:44:48 INFO backup.SnapshotS3Util: Snapshot name   : test1-snapshot-20140828_154448
14/08/28 15:44:48 INFO backup.SnapshotS3Util: Bucket name     : hbasebucketname
14/08/28 15:44:48 INFO backup.SnapshotS3Util: S3 Path         : /hbase
14/08/28 15:44:48 INFO backup.SnapshotS3Util: HDFS Path       : /hbase
14/08/28 15:44:48 INFO backup.SnapshotS3Util: Snapshot From Url : hdfs://nameservice1/hbase
14/08/28 15:44:48 INFO backup.SnapshotS3Util: Mappers         : 4
14/08/28 15:44:48 INFO backup.SnapshotS3Util: s3 protocol     : s3n://
14/08/28 15:44:48 INFO backup.SnapshotS3Util: Snapshot TTL    : 0
14/08/28 15:44:48 INFO backup.SnapshotS3Util: --------------------------------------------------
14/08/28 15:44:48 INFO backup.SnapshotS3Util: Creating snapshot...
14/08/28 15:45:16 INFO snapshot.ExportSnapshot: Export Completed: test1-snapshot-20140828_154448
14/08/28 15:45:16 INFO backup.SnapshotS3Util: Successfully exported snapshot 'test1-snapshot-20140828_154448' to S3
14/08/28 15:45:16 INFO backup.SnapshotS3Util: Complete

```
Import from s3n:// table backup in to Hbase

```
hbase shell
delete_snapshot 'test1-snapshot-20140828_154448'
scan 'test1'
exit

Ensure hbase user can write to HDFS /user

sudo -u hdfs hadoop fs -chmod 777 /user

Import snapshot from s3n store directory /hbase (-p) to HDFS /hbase (-d)

sudo -u hbase HADOOP_CLASSPATH=$(hbase classpath) hadoop jar target/snapshot-s3-util-2.0.1.jar com.imgur.backup.SnapshotS3Util --import --snapshot test1-snapshot-20140828_154448 -d /hbase -p /hbase --awsAccessKey <accessKey> --awsAccessSecret <accessSecret> --bucketName <bucketName> --mappers <numMaps>

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
14/08/28 15:48:58 INFO backup.SnapshotS3Util: Mappers         : 4
14/08/28 15:48:58 INFO backup.SnapshotS3Util: s3 protocol     : s3n://
14/08/28 15:48:58 INFO backup.SnapshotS3Util: Snapshot TTL    : 0
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
usage: BackupUtil [-a] -b <arg> -c | -e | -f <arg> | -i | -x [-d <arg>]   -k <arg>
       [-l <arg>] [-m <arg>] [-n <arg>] [-p <arg>] -s <arg> [-t <arg>]
Backup utility for creating snapshots and exporting/importing to and from
S3
 -a,--s3                      Use s3 protocol instead of s3n. Currently not
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
 -m,--mappers <arg>           The number of parallel copiers if copying
                              to/from S3. Default: 1
 -n,--snapshot <arg>          The snapshot name. Required for importing
                              from S3
 -p,--s3Path <arg>            The snapshot directory in S3. Default is
                              '/hbase'
 -s,--awsAccessSecret <arg>   The AWS access secret string
 -t,--table <arg>             The table name to create a snapshot from.
                              Required for creating a snapshot
 -x,--createExport            Create HBase snapshot AND export to S3

```

##Required HDFS core-site.xml 

```
<!-- Amazon S3 -->
<property>
    <name>fs.s3.awsAccessKeyId</name>
    <value>key</value>
</property>
<property>
    <name>fs.s3.awsSecretAccessKey</name>
    <value>secret</value>
</property>
<!-- Amazon S3N -->
<property>
  <name>fs.s3.impl</name>
  <value>org.apache.hadoop.fs.s3.S3FileSystem</value>
</property>
<property>
  <name>fs.s3n.impl</name>
  <value>org.apache.hadoop.fs.s3native.NativeS3FileSystem</value>
</property>
<property>
  <name>fs.AbstractFileSystem.s3.impl</name>
  <value>org.apache.hadoop.fs.s3.S3FileSystem</value>
</property>
<property>
  <name>fs.AbstractFileSystem.s3n.impl</name>
  <value>org.apache.hadoop.fs.s3native.NativeS3FileSystem</value>
</property>
<property>
    <name>fs.s3n.awsAccessKeyId</name>
    <value>key</value>
</property>
<property>
    <name>fs.s3n.awsSecretAccessKey</name>
    <value>secret</value>
</property>
<property>
    <name>fs.s3n.multipart.uploads.enabled</name>
    <value>true</value>
</property>
<property>
    <name>fs.s3n.multipart.uploads.block.size</name>
    <value>67108864</value>
</property>
<property>
    <name>fs.s3n.multipart.copy.block.size</name>
    <value>5368709120</value>
</property>
```

##Required jets3t.properties

Create a ~/jets3t.properties file on the HBase gateway nodes (where you run this s3 util from)
Copy to /etc/hadoop/conf

```
storage-service.internal-error-retry-max=5
storage-service.disable-live-md5=false
threaded-service.max-thread-count=20
threaded-service.admin-max-thread-count=20
s3service.max-thread-count=20
s3service.admin-max-thread-count=20
```

Note this file will be removed from /etc/hadoop/conf when Cloudera deploy client config is run so will
need to copy back before running the backup script
