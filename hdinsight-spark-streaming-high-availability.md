---
title: Creating highly available Spark Streaming jobs in YARN - Azure HDInsight | Microsoft Docs
description: ''
services: hdinsight
documentationcenter: ''

tags: azure-portal
keywords: spark, streaming, YARN
---

# Creating highly available Spark Streaming jobs in YARN

Spark Streaming enables you to implement scalable, high-throughput, fault-tolerant applications for the processing of data streams. You can run your Spark Streaming applications on HDInsight Spark clusters, and connect it to process data from a variety of sources such as Azure Event Hubs, Azure IoT Hub, Kafka, Flume, Twitter, ZeroMQ, raw TCP sockets or even by monitoring the HDFS filesystem for changes. 

Spark Streaming creates a long running job during which you are able to apply transformations to the data (such as map, reduce, join and extract data by windows of time) and then push out the results to filesystems, databases, dashboards and the console. 

![Stream Processing with HDInsight and Spark Streaming ](./media/hdinsight-spark-streaming-high-availability/hdinsight-spark-streaming.png)

Spark Streaming takes a micro-batch approach to how it processes data. This means that it must first wait to collect a time-defined batch of events (usually configred in the range of less than a second to a few seconds), before it sends the batch of events on for processing. This is in contrast to approaches that would receive a single event and process that single event immediately. The benefit of the micro-batch approach, as you will see, is it lets more efficiently process data that is rapidly ingested into your solution and gives you an abstraction that makes applying aggregate calculations on the events a little easier. 

The design goals of Spark Streaming include low latency (measured in seconds) and linear scalability. However, what sets Spark Streaming apart are its support for fault tolerance with the guarantee that any given event would be processed exactly once, even in the face of a node failure. Additionally, Spark Streaming is integrated with the Spark core API, giving Spark developers a familiar programming model and new developers one less new framework to learn when first starting with Spark. 

Spark Streaming represents a continous stream of data using a discretized stream or DStream. A DStream can be thought of as a series of RDDs.  This DStream can be created from input sources like Event Hubs or Kafka, or by applying transformation on another DStream.  The DStream represents a few layers of abstraction on top of the raw event data. To understand how they work, it helps to build up a DStream from a single event. 

Start with a single event, say a temperature reading from a connected thermostat. When this event arrives at your Spark Streaming application, the first thing that happens is the event is stored in a reliable way- it is replicated so that multiple nodes have a copy of your event. This ensures that the failure of any single node will not result in the loss of your event. Spark core has a data structure that distributes data across multiple nodes in the cluster, where each node generally maintains its data completely in-memory for best performance. This data structure is called a resilient distributed dataset or RDD. The temperature reading event will be stored in an RDD. 

Each RDD represents events collected over some user defined timeframe called the batch interval. Everytime the batch interval elapses a new RDD is produced that contains all the data in the interval of time that just completed. It is this continuous set of RDD's that are collected into a DStream. So for example, if the batch interval was configured to be 1 second long, your DStream emits a batch every second containing one RDD that contains all the data ingested during that second. When processing the DStream, the temperature event would appear in one of these batches. A Spark Streaming application that processes these events, processes the batches that contains the events and ultimately acts on the data stored in the RDD each batch contains. 

---

### Fault-tolerance and RDDs

RDDs have a number of properties that aid the ability to create highly-available and fault tolerant Spark Streaming Jobs.  These properties are the following:

* Batches of input data stored in RDDs as a DStream, are replicated in memory for fault-tolerance.
* Data lost due to worker failure can be recomputed from replicated input data on different workers
* Fast Fault Recovery - recovers from faults/stragglers within 1 second because computation happens in memory
* All transformations are fault-tolerant and exactly-once transformation

![RDD Fault tolerance](./media/hdinsight-spark-streaming-high-availability/fault-tolerance.png)

![Example DStream with Temperature Events ](./media/hdinsight-spark-streaming-high-availability/hdinsight-spark-streaming-example.png)

## Structure of a Spark Streaming Application
A Spark Streaming application is a long running application that receives data from ingest sources, applies transformation to process the data and then pushes the data out to one or more destinations. The structure of a Spark Streaming application has two main parts. First, you define the processing logic that includes where the data comes from, what processing to do on the data and where the results should go. Second, you run the defined application indefinitely, waiting for any signals to stop the long running application.  To understand the structure of a Spark Streaming Application, we will show a simple application that receives a line of text over a TCP socket and counts the number of times each word appears. 

### Define the application
The application definition consists of four main steps: creating a StreamingContext, creating a DStream from the StreamingContext, applying transformations to the DStream and ouputting the results. During this phase you are just describing the application logic, no actual transformations are applied or is output emitted until you run the application (as shown in the second phase).

#### Create a StreamingContext
Create a StreamingContext from the SparkContext that points to your cluster. When creating a StreamingContext you specify the size of the batch in seconds. In this example, we create the StreamingContext so it has a batch size of one second.

    val ssc = new StreamingContext(spark, Seconds(1))

#### Create a DStream
Using the StreamingContext instance you created, create an input DStream for your input source. In this case, we are opening watching for the appearance of new files in default storage attached to the HDInsight cluster. 

    val lines = ssc.textFileStream("/uploads/2017/01/")

#### Apply transformations
You implement the processing by applying transformations on the DStream. Our application will receive one line of text at a time from the file, split each line into words, and then follows the map reduce pattern to count the number of times each word appears.

    val words = lines.flatMap(_.split(" "))
    val pairs = words.map(word => (word, 1))
    val wordCounts = pairs.reduceByKey(_ + _)

#### Output results
Push the transformation results out to the destination systems by applying output operations. In this case, we show the result of each run thru the computation in the console output. 

    wordCounts.print()

### Run the application
Start the streaming application and run until a termination signal is received. 

    ssc.start()            
    ssc.awaitTermination()

For details on the Spark Stream API, along with the event sources, transformations and output operations it supports see [Spark Streaming Programming Guide](https://people.apache.org/~pwendell/spark-releases/latest/streaming-programming-guide.html).

Here is another sample application that is completely self-contained that you can run inside a [Jupyter Notebook](hdinsight-apache-spark-jupyter-notebook-kernels.md). In the example below, we create a mock data source in the class DummySource that outputs the value of a counter and the current time in milliseconds every 5 seconds. We create a new StreamingContext object that has a batch interval of 30 seconds. Every time a batch is created, it examines the RDD produced, converts it to a Spark DataFrame and creates a temporary table over the DataFrame. 

    class DummySource extends org.apache.spark.streaming.receiver.Receiver[(Int, Long)](org.apache.spark.storage.StorageLevel.MEMORY_AND_DISK_2) {

        /** Start the thread that simulates receiving data */
        def onStart() {
            new Thread("Dummy Source") { override def run() { receive() } }.start()
        }

        def onStop() {  }

        /** Periodically generate a random number from 0 to 9, and the timestamp */
        private def receive() {
            var counter = 0  
            while(!isStopped()) {
            store(Iterator((counter, System.currentTimeMillis)))
            counter += 1
            Thread.sleep(5000)
            }
        }
    }

    // A batch is created every 30 seconds
    val ssc = new org.apache.spark.streaming.StreamingContext(spark.sparkContext, org.apache.spark.streaming.Seconds(30))

    // Set the active SQLContext so that we can access it statically within the foreachRDD
    org.apache.spark.sql.SQLContext.setActive(spark.sqlContext)

    // Create the stream
    val stream = ssc.receiverStream(new DummySource())

    // Process RDDs in the batch
    stream.foreachRDD { rdd =>

        // Access the SQLContext and create a table called demo_numbers we can query
        val _sqlContext = org.apache.spark.sql.SQLContext.getOrCreate(rdd.sparkContext)
        _sqlContext.createDataFrame(rdd).toDF("value", "time")
            .registerTempTable("demo_numbers")
    } 

    // Start the stream processing
    ssc.start()

We can then query the DataFrame periodically to see the current set of values present in the batch. In this case, we use the following SQL query. 

    %%sql
    SELECT * FROM demo_numbers

The resulting output looks similar to the following:

| value	| time |
| --- | --- |
|10 | 1497314465256 |
|11 | 1497314470272 |
|12	| 1497314475289 |
|13	| 1497314480310 |
|14	| 1497314485327 |
|15	| 1497314490346 |

In the above expect six values in the typical case, because the DummySource creates a value every 5 seconds, and we emit a batch every 30 seconds.

## Sliding Windows
If you want to perform aggregate calculations on your DStream over some time period, for example to get an average temperature over the last 2 seconds, you can use the sliding window operations included with Spark Streaming. A sliding window is defined as having a duration (referred to as the window length) and the interval at which the window's content are evaluated (referred toas the slide interval). 

These sliding windows can overlap, for example you can define a window with a length of 2 seconds, that slides every 1 second. This means every time you perform an aggregation calculation, the window will include data from the last 1 second of the previous window as well as any new data in the next 1 second. 

![Example Initial Window with Temperature Events ](./media/hdinsight-spark-streaming-high-availability/hdinsight-spark-streaming-window-01.png)

![Example Window with Temperature Events After Sliding](./media/hdinsight-spark-streaming-high-availability/hdinsight-spark-streaming-window-02.png)

By way of example, we can enhance the code that uses the DummySource above to first collect the batches into a window with a 1 minute duration, that slides by 1 minute as well.

    // A batch is created every 30 seconds
    val ssc = new org.apache.spark.streaming.StreamingContext(spark.sparkContext, org.apache.spark.streaming.Seconds(30))

    // Set the active SQLContext so that we can access it statically within the foreachRDD
    org.apache.spark.sql.SQLContext.setActive(spark.sqlContext)

    // Create the stream
    val stream = ssc.receiverStream(new DummySource())

    // Process batches in 1 minute windows
    stream.window(org.apache.spark.streaming.Minutes(1)).foreachRDD { rdd =>

        // Access the SQLContext and create a table called demo_numbers we can query
        val _sqlContext = org.apache.spark.sql.SQLContext.getOrCreate(rdd.sparkContext)
        _sqlContext.createDataFrame(rdd).toDF("value", "time")
        .registerTempTable("demo_numbers")
    } 

    // Start the stream processing
    ssc.start()

After the first minute, this will yield 12 entries or six entries from each of the two batches collected in the window.

| value	| time |
| --- | --- |
| 1 |	1497316294139 |
| 2 |	1497316299158
| 3	| 1497316304178
| 4	| 1497316309204
| 5	| 1497316314224
| 6	| 1497316319243
| 7	| 1497316324260
| 8	| 1497316329278
| 9	| 1497316334293
| 10 | 1497316339314
| 11 | 1497316344339
| 12 | 1497316349361

The sliding window functions available in the Spark Streaming API include window, countByWindow, reduceByWindow and countByValueAndWindow. For details on these functions see [Transformations on DStreams](https://people.apache.org/~pwendell/spark-releases/latest/streaming-programming-guide.html#transformations-on-dstreams).

## Checkpointing
In order to deliver resiliency and fault tolerance, Spark Streaming relies on checkpointing to insure that stream processing can continue uninterrupted, even in the face of node failures. In HDInsight, Spark creates checkpoints to durable storage (Azure Storage or Data Lake Store). These checkpoints store the metadata about the streaming application- such as the configuration, the operations defined by the application and any batches that were queued but not yet processed. In some cases, the checkpoints will also include the saving of the data in the RDD's to shorten the time it takes to rebuild the state of the data from what is presend in the RDD's managed by Spark. 

---
## About Spark Structured Streaming

A new component in the Spark ecosystem is Structured Spark Streaming.  It was introduced in Spark 2.0.  Structured Streaming is a an analytic engine
for streaming structured data.  It shares the same set of APIs with the SparkSQL batching engine.  Structured Streaming provides the ability
to run computation over continuously arriving data and to keep updating results.  It uses the Spark micro-batches streaming model. You use the `getOffset` and the `getBatch` APIs along with user-defined operations to process the data using SparkSQL syntax.  Then you use a sink and `addBatch` to be able to persist the processed DataFrame.

## Streams as Tables
Spark Structured Streaming takes the perspective that a stream of data can be represented as a table that is unbounded in height- in other words it continues to grow as new data arrives. This Input Table is continously processed by a long running query, and the results flushed out to an Output Table. This concept is best explained with the following illustration:

![Structured Streaming Concept](./media/hdinsight-spark-structured-streaming-overview/hdinsight-spark-structured-streaming-concept.png)

In Structured Streaming data arrives at the system and is immediately ingested into an Input Table. You write queries (using the DataFrame and Dataset APIs) that perform operations against this Input Table. The query output yields another table, called the Results Table. The Results Table contains results of your query from which you draw any data you would send to an external datastore (such a relational database). The timing of when data is processed from the Input Table is controlled by the trigger interval. By default Structured Streaming tries to process the data as soon as it arrives. In practice this means as soon as it is done processing the run of the previous query, it starts another processing run against any newly received data. However, you can also configure the trigger to run on a longer interval, so that the streaming data is processed according to time-based batches. 

The output mode controls the data in the Results Tables. This data may be completely refreshed everytime there is new data so that it includes all of the output data since the streaming query began (called `complete mode`), or it may only contain just the data that is new since the last time the query was processed (called `append mode`).

## Checkpointing and Write Ahead Logs
In order to deliver resiliency and fault tolerance, Structured Streaming relies on checkpointing to insure that stream processing can continue uninterrupted, even in the face of node failures. In HDInsight, Spark creates checkpoints to durable storage (Azure Storage or Data Lake Store). These checkpoints store the progress information about the streaming query. In addition, Structured Streaming utilizes a Write Ahead Log. The purpose of the WAL is to capture ingested data that has been received but not processed by a query, so if a failure occurs and processing is restarted the events received from the source are not lost.

---
## About Spark Streaming and YARN

XXXXX - add new content here

### 4 Problems Made when writing streaming applications

What can fail - an executor or a driver.  Some failures and guarantee requirements needs additonal configurations and setups.  Also many streaming 
applications need ZERO data loss guarantees in spite of any type of error that could occur in the streaming system.

If an executor fails, then tasks and receivers are restarted by Spark automatically, there is no configuration change needed.  If a driver
fails, then all of executors fail.  Also all computation and received blocks are lost.  Use DStream Checkpointing to recover from driver failure.  
DStream Checkpointing periodically saves the DAG of DStreams to fault-tolerant storage (such as Azure BLOBs).  Checkpointing allows for 
the failed driver to be restarted using checkpoint information.  This driver restart will launch new executors and will also restart
receivers.

To recover drivers w/DStream Checkpointing
1. Configure automatic driver restart on the cluster manager (YARN)
2. Set a check point directory in a HDFS-compatible file system by `streamingContext.checkpoint(hdfsDirectory)`
3. Restructure source code to use checkpoints for recovery.
4. Configure to recover lost data

To Configure automatic driver restart in YARN
* In YARN use config `yarn.resourcemanager.am.max-attempts` setting

To Restructure code to use checkpoints
* Put all setup code into a function that returns a new StreamingContext

```scala
    def creatingFunc() : StreamingContext = {
        val context = new StreamingContext(...)
        val lines = KafkaUtils.createStream(...)
        val words = lines.flatMap(...)
        ...
        context.checkpoint(hdfsDir)
    }

    val context = StreamingContext.getOrCreate(hdfsDir, creatingFund)
    conext.start()
```
To recover lost data blocks in a restarted driver
* Use a `Write Ahead Log` (or WAL) which synchronously saves received data to fault-tolerant storage
* Requires that checkpointing with logs written to checkpoint directory be enabled
* Enable WAL via `sparkConf.set("spark.streaming.receiver.writeAheadLog.enable","true")`
* Disable in-memory replication via `StorageLevel.MEMORY_AND_DISK_SER` for input DStreams

At least once, w/Checkpointing + WAL + reliable receivers
Exactly once, as long as received data is not lost and if outputs are idempotent or transactional
Exactly once (alternative) via new Kafka Direct approach - uses Kafka as replicated log, does not use receivers or WALs

---

1. Not monitoring and managing your streaming jobs - WHY: because it's so much harder for streaming than for batch jobs. Spark streaming is long-running.  YARN doesn't aggreage logs until job finishes.  Spark checkpoints can't survive app or Spark upgrades.  Need to clear checkpoint directory during upgrade.  Use YARN Cluster mode to run drivers even if client fails.  Set up automatic restart on driver, use spark configurion 

```
    spark.yarn.maxAppAttempts = 2
    spark.yarn.am.attemptFailuresValidityInterval=1h
```
Monitoring - Spark Streaming UI AND Spark has a configurable metrics system, you can also use Graphite/Grafana to download dashboard metrics such
as 'num records processed', 'memory/GC usage on driver & executors', 'total delay', 'utilization of the cluster' and others...
In Structured Streaming (2.1 or greater only) use `StreamingQueryListener`  

2. Not considering data loss - driver will restart, but...
    * if file, then use checkpointing (means both metadata AND data, but doesn't work across Spark upgrades for legacy streaming, it does upgrade ok if you are using Structured Streaming)
    * if receiver (Kafka), then enable checkpointing ADN enable the write-ahead log set `spark.streaming.receiver.writeAheadLog.enable` = true [needed as data is stored in executor memory only by default, this makes a copy of the data in a durable location, but this can slow down the job, or Kafka direct stream, but for Kafka don't use WAL, rather use Kafka's own data storage [it can replay the data from it's own offsets], but then you have to manage those offsets (maybe in a database) 
    * if ???, then  

3. Counting - no dupliates (exactly-once semantics / perfect counting) - double submits are the current big problem (w/Kafka) fix by adding sequence IDs to sources (fails beyond a billion submits)  

4. Not shutting down your streaming app gracefully - offsets known, state stored externally, stopping at the right place - when / how to kill a YARN (Spark streaing) app gracefully?  Use 'thread hooks' in Spark, check for external flag every n seconds -or- use a marker file, that is 
touch a file when starting the app on HDFS, remove the file when you want to stop, use a separate thread in Spark app, then calls 
```scala
    streamingContext.stop(stopSparkContext = true, stopGracefully = true)
```
store offsets externally in an external database, to be able to recover on restart


PROBLEM: YARN aggregates logs only on job completion (doesn't work well for Streaming), so you can collect logs and make them searchable in real-time using other services, such as Logstash, Elasticsearch and Kibana.  Configure Log4j to write into Logstash.  For resource monitoring / altering you often add an additional logging solution, such as Graphite logging into InfluxDB and out into Graphana

---
This article focuses on how you configure Spark Streaming to achieve exactly once processing semantics in a highly-available, fault-tolerant way.

## Event Process Approaches

Stream processing applications may take different approaches to how they handle the re-processing of messages in the face of some failure in the system, these approaches yield different event processing semantics:

* **At least once**: In some cases, it is acceptable for a given event to be processed multiple times. If the system guarantees that the message will always get processed, but it may get processed more than once, than it provides an at least once semantics. 
* **At most once**: In some cases, it is acceptable to lose messages, but it is critical that messages not be processed more than once. A system to supports this is providing an at most once processing semantics.
* **Exactly once**: In other cases, a message must processed once and only once irrespective of failures in components of the system. Systems that support this provide an exactly once semantics. 

NOTE: This article focuses on how you configure Spark Streaming to achieve exactly once processing semantics. 

---

## Achieving Exactly Once Semantics with Spark Streaming
To achieve exactly once processing of an event by a Spark Streaming application, you need to consider how all of the points of failure restart after having an issue and how you can avoid data loss. Consider that in a Spark Streaming application you have an input source, a driver process that manages the long running job, one or more receiver processes that pull data from the input source, and tasks that apply the processing and push the results to an output sink. To achieve exactly once semantics means ensuring that no data is lost at any point and that the processing is restartable, regadless of where the failure occurs. The following describes each of the key components of achieving exactly once semantics.

### Replayable Sources & Reliable Receivers
The source your Spark Streaming application is reading your events from must be replayable. This means that it should be possible to ask the source to provide the message again in cases where the message was retrieved but then the system failed before it could be persisted or processed. In Azure,  both Azure Event Hubs and Kafka on HDInsight provide replayable sources. An even simpler example of a replayable source is a fault-tolerant file system like HDFS, Azure Storage blobs or Azure Data Lake Store where all the data can be kept in perpetuity (and you can re-read it in its entirety at any point). In Spark Streaming, sources like Event Hubs and Kafka have reliable receivers, meaning they keep track of their progress reading thru the source and persist it to fault-tolerant storage (either within ZooKeeper or in Spark Streaming checkpoints written to HDFS). That way, if a receiver fails and is later restarted it can pick up where it left off.    

### Use the Write Ahead Log
Spark Streaming supports the use of a Write Ahead Log, where any event received from a source is first written to Spark's checkpoint directory in fault-tolerant storage (in Azure this is HDFS backed by either Azure Storage or Azure Data Lake Store) before being stored in an RDD by the receiver running within a worker. In your Spark Streaming application, the Write Ahead Log is enabled for all receivers by setting the ```spark.streaming.receiver.writeAheadLog.enable``` configuration to ```true```. The Write Ahead Log provides fault-tolerance for failures of both the driver and the executors.  

In the case of workers running tasks against the event data, consider that following the insertion into the Write Ahead Log, the event is inserted by the receiver into an RDD, which by definition is both replicated and distributed across multiple workers. Should the task fail because the worker running it crashed, the task will simply be restarted on another worker that has a replica of the event data, so the event is not lost. 

### Use Checkpoints
The drivers need to be restartable. If the driver running your Spark Streaming application crashes, it takes down with it all executors running receivers, tasks and any RDD's storing event data. This surfaces two considerations- first you need to be able to save the progress of the job so you can resume it later. This is accomplished by checkpointing the Directed Acyclic Graph (DAG) of the DStream periodically to fault-tolerant storage. This metadata includes the configuration used to create the streaming application, the operations that define the application, and any batches that are queued but not yet completed. This will enable a failed driver to be restarted from the checkpoint information. When it restarts, it will launch new receivers that themselves recover the event data back into RDD's from the Write Ahead Log. Checkpoints are enabled in Spark Streaming in two steps. On the StreamingContext object you configure the path in storage to where the checkpoints are stored:

    val ssc = new StreamingContext(spark, Seconds(1))
    ssc.checkpoint("/path/to/checkpoints)

In HDInsight, these checkpoints should be saved to your default storage attached to your cluster (either Azure Storage or Azure Data Lake Store). Next, you need to specify a checkpoint interval (in seconds) on the DStream that controls how often any state data (e.g., state derived from the input event) is persisted to storage. Persisting state data this way can reduce the computation needed when rebuilding the state from the source event. 

    val lines = ssc.socketTextStream("hostname", 9999)
    lines.checkpoint(30)
    ssc.start()
    ssc.awaitTermination()

### Use Idempotent Sinks
The destination to which your job writes results must be able to handle the situation where it is given the same result more than once. It must be able to detect such duplicate results and ignore them. You can achieve idempotent sinks by implementing logic that first checks for the existence of the result in the datastore. If the result already exists, the write should appear to succeed from the perspective of your Spark job, but in reality your data store ignored the duplicate data. If the result does not exist, then it should insert the new result into its storage. One example of this is to use a stored procedure with Azure SQL Database that is used to insert events into a table. When the stored procedure is invoked, it first looks up the event by key fields and only if none are found is the record inserted into the table. Another example is to use a partitioned file system, like Azure Storage blobs or Azure Data Lake store. In this case your sink logic does not need to check for the existence of a file. If the file representing the event exists, it is simply overwritten with the same data. Otherwise, a new file is created at the computed path. In the end idempotent sinks should support being called multiple times with the same data and no change of state should result. 

### Use YARN Capacity Scheduling

When a Spark Streaming application is submitted to the cluster, YARN queue where the job runs must be defined. I strongly recommend using [YARN Capacity Scheduler](https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/CapacityScheduler.html) and submitting long-running jobs to separate queue.
---

## Next steps
This article covered the key components of achieving exactly once semantics in a fault-tolerant and highly-available way with your Spark Streaming applications. 

- [Spark Streaming Overview](hdinsight-spark-streaming-overview) 
- [Long-running Spark Streaming Jobs on YARN](http://mkuthan.github.io/blog/2016/09/30/spark-streaming-on-yarn/) 
- [Discretized Streams: A Fault-Tolerant Model for Scalable Stream Processing](https://www2.eecs.berkeley.edu/Pubs/TechRpts/2012/EECS-2012-259.pdf)
