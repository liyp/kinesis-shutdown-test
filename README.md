Test for Kinesis Stream shutdown hook
=====================================

我线上的疑问：
> 我的Consumers程序使用 KCL 方式读取并处理流中数据， RecordProcessor 实现了 IRecordProcessor接口，其中 RecordProcessor.shutdown 方法中调用 checkpoint() ，以实现程序重启时能从上次退出时的处理点开始继续处理流中记录。 可实际情况是会重复处理上次退出时处理了的记录。
> 
> 我打开日志发现，RecordProcessor.shutdown() 并没有没调用过。 我在程序中注册了 Shutdown Hook，并调用了 Worker.shutdown() ，但在kill 后，日志中也只有 "Worker xxx  has successfully stopped lease-tracking threads" 。RecordProcessor并没有正常地shutdown，进程便退出了。
> 
> 请问，如何让 RecordProcessor 执行 shutdown？ 我的目的是，当Consumer程序退出后，下次启动能从上次退出点开始处理记录，而不会重复处理。

### 我的测试步骤

为了方面讨论，我直接使用`aws-sdk-java`中的 [kinesis samples](https://github.com/aws/aws-sdk-java/tree/master/src/samples/AmazonKinesis)，做如下修改：

*  AmazonKinesisApplicationSample中，使用器（也就是流的消费者）启动前（调用`worker.run()`前）注册Shutdown Hook

```java
// add a shutdown hook to stop the server
Runtime.getRuntime().addShutdownHook(new Thread() {
        @Override
        public void run() {
                System.out.println("Inside Add Shutdown Hook");
                worker.shutdown();
                try {
                        Thread.sleep(10000); // sleep 10s
                } catch (InterruptedException e) {
                        e.printStackTrace();
                }
        }
});
System.out.println("Shut Down Hook Attached.");
```

那么我启动 Kinesis Application

```
mvn clean compile
mvn exec:java -Dexec.mainClass="awskinesis.AmazonKinesisApplicationSample"
```

终止 Kinesis Application 通过 `kill -15 pid` 屏幕输出如下

```
ubuntu@ip-172-31-43-116:~/kinesis/kinesis-shutdown-test$ mvn exec:java -Dexec.mainClass="awskinesis.AmazonKinesisApplicationSample"
[INFO] Scanning for projects...
[INFO] Searching repository for plugin with prefix: 'exec'.
[INFO] ------------------------------------------------------------------------
[INFO] Building kinesis-shutdown-test
[INFO]    task-segment: [exec:java]
[INFO] ------------------------------------------------------------------------
[INFO] [exec:java {execution: default-cli}]
2015-09-16 03:29:19,744 | INFO  | awskinesis.AmazonKinesisApplicationSample.main() | LeaseCoordinator.java:108 | With failover time 10000ms and epsilon 25ms, LeaseCoordinator will renew leases every 3308ms and take leases every 20050ms
Shut Down Hook Attached.
Running SampleKinesisApplication to process stream myFirstStream as worker ip-172-31-43-116.ec2.internal:e3dfd740-7603-42bf-ab48-eb5166fc5ff2...
2015-09-16 03:29:19,747 | INFO  | awskinesis.AmazonKinesisApplicationSample.main() | Worker.java:372 | Initialization attempt 1
2015-09-16 03:29:19,748 | INFO  | awskinesis.AmazonKinesisApplicationSample.main() | Worker.java:373 | Initializing LeaseCoordinator
2015-09-16 03:29:20,427 | INFO  | awskinesis.AmazonKinesisApplicationSample.main() | LeaseManager.java:121 | Table SampleKinesisApplication already exists.
2015-09-16 03:29:20,465 | INFO  | awskinesis.AmazonKinesisApplicationSample.main() | Worker.java:376 | Syncing Kinesis shard info
2015-09-16 03:29:20,697 | INFO  | awskinesis.AmazonKinesisApplicationSample.main() | Worker.java:387 | Starting LeaseCoordinator
2015-09-16 03:29:30,723 | INFO  | awskinesis.AmazonKinesisApplicationSample.main() | Worker.java:319 | Initialization complete. Starting worker loop.
2015-09-16 03:29:30,806 | INFO  | cw-metrics-publisher | DefaultCWMetricsPublisher.java:65 | Successfully published 16 datums.
2015-09-16 03:29:40,823 | INFO  | LeaseCoordinator-2 | LeaseTaker.java:349 | Worker ip-172-31-43-116.ec2.internal:e3dfd740-7603-42bf-ab48-eb5166fc5ff2 saw 1 total leases, 1 available leases, 1 workers. Target is 1 leases, I have 0 leases, I will take 1 leases
2015-09-16 03:29:40,832 | INFO  | cw-metrics-publisher | DefaultCWMetricsPublisher.java:65 | Successfully published 4 datums.
2015-09-16 03:29:40,864 | INFO  | LeaseCoordinator-2 | LeaseTaker.java:166 | Worker ip-172-31-43-116.ec2.internal:e3dfd740-7603-42bf-ab48-eb5166fc5ff2 successfully took 1 leases: shardId-000000000000
2015-09-16 03:29:41,731 | INFO  | awskinesis.AmazonKinesisApplicationSample.main() | Worker.java:535 | Created new shardConsumer for : ShardInfo [shardId=shardId-000000000000, concurrencyToken=c266a306-ef2d-494c-aaa9-c5c52eb440ad, parentShardIds=[]]
2015-09-16 03:29:41,733 | INFO  | pool-2-thread-1 | BlockOnParentShardTask.java:83 | No need to block on parents [] of shard shardId-000000000000
2015-09-16 03:29:42,760 | INFO  | pool-2-thread-1 | KinesisDataFetcher.java:89 | Initializing shard shardId-000000000000 with 49554490746165447495069456200810352230468826796694437890
2015-09-16 03:29:42,945 | INFO  | pool-2-thread-1 | AmazonKinesisApplicationSampleRecordProcessor.java:56 | Initializing record processor for shard: shardId-000000000000
2015-09-16 03:29:50,872 | INFO  | cw-metrics-publisher | DefaultCWMetricsPublisher.java:65 | Successfully published 20 datums.
2015-09-16 03:29:50,905 | INFO  | cw-metrics-publisher | DefaultCWMetricsPublisher.java:65 | Successfully published 12 datums.
2015-09-16 03:30:00,926 | INFO  | cw-metrics-publisher | DefaultCWMetricsPublisher.java:65 | Successfully published 13 datums.
2015-09-16 03:30:10,984 | INFO  | cw-metrics-publisher | DefaultCWMetricsPublisher.java:65 | Successfully published 18 datums.
2015-09-16 03:30:19,776 | INFO  | awskinesis.AmazonKinesisApplicationSample.main() | Worker.java:530 | Current stream shard assignments: shardId-000000000000
2015-09-16 03:30:19,776 | INFO  | awskinesis.AmazonKinesisApplicationSample.main() | Worker.java:530 | Sleeping ...
2015-09-16 03:30:21,015 | INFO  | cw-metrics-publisher | DefaultCWMetricsPublisher.java:65 | Successfully published 13 datums.
Inside Add Shutdown Hook
2015-09-16 03:30:25,779 | INFO  | awskinesis.AmazonKinesisApplicationSample.main() | Worker.java:362 | Stopping LeaseCoordinator.
2015-09-16 03:30:25,780 | INFO  | awskinesis.AmazonKinesisApplicationSample.main() | LeaseCoordinator.java:239 | Worker ip-172-31-43-116.ec2.internal:e3dfd740-7603-42bf-ab48-eb5166fc5ff2 has successfully stopped lease-tracking threads
Main thread exits.
2015-09-16 03:30:31,048 | INFO  | cw-metrics-publisher | DefaultCWMetricsPublisher.java:65 | Successfully published 18 datums.
```

 `AmazonKinesisApplicationSampleRecordProcessor.shutdown()`被调用会打印 `Shutting down record processor for shard `，而日志中未出现，哪时应该被调用呢？

在[官方文档](http://docs.aws.amazon.com/zh_cn/kinesis/latest/dev/kinesis-record-processor-implementation-app-java.html)中，有这样的描述

> **shutdown**  
> `public void shutdown(IRecordProcessorCheckpointer checkpointer, ShutdownReason reason)`  
> KCL 在处理结束（关闭原因为 TERMINATE）或工作程序不再响应（关闭原因为 ZOMBIE）时调用 shutdown 方法。  
> 处理操作在记录处理器不再从分片中接收任何记录时结束，因为分片已被拆分或合并，或者流已删除。  
> KCL 还会将 IRecordProcessorCheckpointer 接口传递到 shutdown。如果关闭原因为 TERMINATE，则记录处理器应完成处理任何数据记录，然后对此接口调用 checkpoint 方法。  
 
