---
title: Changing Configs via Ambari - Azure HDInsight | Microsoft Docs
description: ''
services: hdinsight
documentationcenter: ''

tags: azure-portal
keywords: ambari,config,hive,pig,hadoop

---
# Changing Configs via Ambari

## Overview

HDInsight allows for the creation of Apache Hadoop clusters for large-scale data processing applications. Managing and monitoring multinode complex clusters is a tedious job. [Apache Ambari](http://ambari.apache.org/) is a web interface to easily manage and monitor HDInsight Linux clusters. The Ambari web interface is only available with Linux clusters. For Windows clusters, the Ambari [REST API](hdinsight-hadoop-manage-ambari-rest-api) can be used.

In this article, you’ll learn how to use the Ambari web user interface to manage and optimize configurations of an HDInsight Linux cluster.

For an introduction to using the Ambari Web UI, take a look at [Manage HDInsight clusters by using the Ambari Web UI](hdinsight-hadoop-manage-ambari)

When you log in to its web interface (HTTPS://CLUSTERNAME.azurehdidnsight.net), Ambari displays a useful dashboard that gives you a great overview of your cluster at-a-glance.

![Ambari dashboard](./media/hdinsight-changing-configs-via-ambari/ambari-dashboard.png)

The Ambari web UI can be used to manage hosts, services, alerts, configurations, and views. It can’t be used to create an HDInsight cluster, upgrade services, manage stacks and versions, manage users, groups, and permissions, decommission or recommission hosts, or add services to the cluster.

## Manage your cluster's configuration

Configuration settings help tune a particular service. To modify the configuration setting of a service, select the service from the **Services** sidebar (on the left), and then navigate to the **Configs** tab in the service detail page.

![Services sidebar](./media/hdinsight-changing-configs-via-ambari/services-sidebar.png)

### Modify NameNode Java heap size

To modify the NameNode Java heap size, follow the steps below:

1. Select **HDFS** from the Services sidebar and navigate to the **Configs** tab.

![HDFS configuration](./media/hdinsight-changing-configs-via-ambari/hdfs-config.png)

2. Find the setting **NameNode Java heap size**. You can also use the **filter** text box to type and find a particular setting. Click the **pen** icon beside the setting name.

![NameNode Java heap size](./media/hdinsight-changing-configs-via-ambari/java-heap-size.png)

3. Type the new value in the text box, and then press **Enter** to save the change.

![Edit NameNode Java heap size](./media/hdinsight-changing-configs-via-ambari/java-heap-size-edit.png)

4. Note that the NameNode Java heap size is changed to 2 GB from 1 GB.

![Edited NameNode Java heap size](./media/hdinsight-changing-configs-via-ambari/java-heap-size-edited.png)

> Note: The NameNode Java heap size depends on many factors such as load on the cluster, number of files, and number of blocks. The default size of 1 GB works well with most clusters, although certain workloads may require modification.

5. Save your changes by clicking on the green **Save** button on the top of the configuration screen.

![Save changes](./media/hdinsight-changing-configs-via-ambari/save-changes.png)

### Hive optimization

As mentioned earlier, each service has certain configuration parameters that can be easily modified using the Ambari web UI. In this section, we’ll learn about important configuration options to optimize overall Hive performance.

1. To modify Hive configuration parameters, select **Hive** from the Services sidebar.
2. Navigate to the **Configs** tab.

### Set the Hive execution engine

There are two execution engines: MapReduce and Tez. Tez is faster than MapReduce. HDInsight Linux clusters have Tez as the default execution engine. To change the execution engine, follow these steps.

1. In the Hive **Configs** tab, type **execution engine** in the filter box.

![Search execution engine](./media/hdinsight-changing-configs-via-ambari/search-execution.png)

2. In the **Optimization** property, observe that the default value is **Tez**.

![Optimization - Tez](./media/hdinsight-changing-configs-via-ambari/optimization-tez.png)

 
### Tune mappers

Hadoop tries to split a single file into multiple files and process the resulting files in parallel. The number of mappers depends on the number of splits. The following two configuration parameters drive the number of splits for the Tez execution engine:

* `tez.grouping.min-size`: Lower limit on the size of a grouped split (default value of 16,777,216 bytes).
* `tez.grouping.max-size`: Upper limit on the size of a grouped split (default value of 1,073,741,824 bytes).

For example, to set four mapper tasks for a data size of 128 MB, you would set both parameters to 32 MB each (33,554,432 bytes).

1. Modify the above configuration parameters by navigating to the **Configs** tab of the Tez service. Expand the General panel, and then locate the `tez.grouping.max-size` and `tez.grouping.min-size` parameters.

2. Set both parameters to **33,554,432** bytes (32 MB).

![Tez grouping sizes](./media/hdinsight-changing-configs-via-ambari/tez-grouping-size.png)
 
> Note: The changes made here will affect all Tez jobs across the server. The parameter values should be carefully modified in order to get the optimal result.


### Tune reducers

The number of reducers is calculated based on the parameter `hive.exec.reducers.bytes.per.reducer`. The parameter specifies the number of bytes processed per reducer. The default value is 64 MB.

1. To modify the parameter, navigate to the Hive **Configs** tab and find the **Data per Reducer** parameter on the Settings page.

![Data per Reducer](./media/hdinsight-changing-configs-via-ambari/data-per-reducer.png)
 
2. Select **Edit** to modify the value to 128 MB (134217728 bytes), and then press **Enter** to save.

![Data per Reducer - edited](./media/hdinsight-changing-configs-via-ambari/data-per-reducer-edited.png)
  
Given an input size of 1,024 MB, with 128 MB of data per reducer, there will be 1024/128, or 8 reducers.

3. An invalid or wrong value for the Data per Reducer parameter may result in a large number of reducers, adversely affecting query performance. To limit the maximum number of reducers, set `hive.exec.reducers.max` to an appropriate value. The default value is 1009.
 
 
### Enable parallel execution

A Hive query is executed in one or more stages. If the independent stages can be run in parallel, this will increase query performance.

1.	To enable parallel query execution, navigate to the Hive **Config** tab and search the `hive.exec.parallel` property. The default value is false. Change the value to **true**, and then press **Enter** to save the value.
 
2.	To limit the number of jobs to be run in parallel, modify the `hive.exec.parallel.thread.number` property. The default value is 8.

![Hive exec parallel](./media/hdinsight-changing-configs-via-ambari/hive-exec-parallel.png)


### Enable vectorization

Hive processes data row by row. Vectorization enables Hive to process data in blocks of 1,024 rows instead of one row at a time.

1. To enable a vectorized query execution, navigate to the Hive **Configs** tab and search for the `hive.vectorized.execution.enabled` parameter. The default value is true for Hive 0.13.0 or later.
 
2. To enable vectorized execution for the reduce side of the query, set the `hive.vectorized.execution.reduce.enabled` parameter to **true**. The default value is false.

![Hive vectorized execution](./media/hdinsight-changing-configs-via-ambari/hive-vectorized-execution.png)
 
> Note: Vectorization is only applicable to the ORC file format.


### Enable cost-based optimization (CBO)

Hive follows a set of rules to find an optimal query execution plan, which represents an old technique. Cost-based optimization evaluates multiple plans to execute a query and assigns a cost to each plan. It then finds the cheapest plan to execute a query.

1. To enable CBO, navigate to the Hive **Configs** tab and filter the `parameter hive.cbo.enable`. Switch the toggle button to **On** to enable CBO.

![CBO](./media/hdinsight-changing-configs-via-ambari/cbo.png)

The following additional configuration parameters increase Hive query performance when CBO is enabled:

`hive.compute.query.using.stats`

When set to **true**, Hive uses stats stored in metastore to answer simple queries like `count(*)`.

![CBO](./media/hdinsight-changing-configs-via-ambari/hive-compute-query-using-stats.png)

`hive.stats.fetch.column.stats`

Column statistics are created when CBO is enabled. Hive uses column statistics, which are stored in metastore, to optimize queries. Fetching column statistics for each column takes longer when the number of columns is high. When set to **false**, this setting disables fetching column statistics from the metastore.

![Hive stats set column stats](./media/hdinsight-changing-configs-via-ambari/hive-stats-fetch-column-stats.png)

`hive.stats.fetch.partition.stats`

Basic partition statistics such as number of rows, data size, and file size are stored in metastore. When set to **true**, the partition stats are fetched from metastore. When false, the file size is fetched from the file system, and the number of rows is fetched from row schema.

![Hive stats set partition stats](./media/hdinsight-changing-configs-via-ambari/hive-stats-fetch-partition-stats.png)


### Enable intermediate compression

Map tasks create intermediate files that are used by the reducer tasks. Intermediate compression shrinks the intermediate file size.

1. To enable intermediate compression, navigate to the Hive **Configs** tab, and then set the `hive.exec.compress.intermediate` parameter to **true**. The default value is false.
 
> Note: To compress intermediate files, choose a compression codec with lower CPU cost, even if it doesn’t have a high compression output.

![Hive exec compress intermediate](./media/hdinsight-changing-configs-via-ambari/hive-exec-compress-intermediate.png)

2. To set the intermediate compression codec, add the custom property `mapred.map.output.compression.codec` to the `hive-site.xml` or `mapred-site.xml` file.

3. To add a custom setting:

    a. Navigate to the Hive **Configs** tab and select the **Advanced** tab.

    b.	Under the Advanced tab, find and expand the **Custom hive-site** pane.

    c.	Click the link **Add Property** at the bottom of the Custom hive-site pane.

    d.	In the Add Property window, enter `mapred.map.output.compression.codec` as the key and `org.apache.hadoop.io.compress.SnappyCodec` as the value.

    e.	Click **Add**.

![Hive custom property](./media/hdinsight-changing-configs-via-ambari/hive-custom-property.png)

This will compress the intermediate file using Snappy compression. Once the property is added, it will appear in the Custom hive-site pane.

> Note: This modifies the `$HADOOP_HOME/conf/hive-site.xml` file.


### Compress final output

The final Hive output can also be compressed.

1. To compress the final Hive output, navigate to the Hive **Configs** tab, and then set the `hive.exec.compress.output` parameter to **true**. The default value is false.
 
2. To choose the output compression codec, add the `mapred.output.compression.codec` custom property to the Custom hive-site pane, as explained above.

![Hive custom property](./media/hdinsight-changing-configs-via-ambari/hive-custom-property2.png)
 

### Enable speculative execution

Speculative execution launches a certain number of duplicate tasks in order to detect and blacklist the slow-running task tracker, while improving the overall job execution by optimizing individual task results.

1. To enable speculative execution, navigate to the Hive **Configs** tab, and then set the `hive.mapred.reduce.tasks.speculative.execution` parameter to **true**. The default value is false.

![Hive mapred reduce tasks speculative execution](./media/hdinsight-changing-configs-via-ambari/hive-mapred-reduce-tasks-speculative-execution.png)
 
> Note: Speculative execution shouldn’t be turned on for long-running MapReduce tasks with large amounts of input.


### Tune dynamic partitions

Hive allows for creating dynamic partitions when inserting records into a table, without predefining each and every partition. This is powerful feature, although it may result in the creation of a large number of partitions and an accordingly large number of files for each partition.

1. For Hive to do dynamic partitions, the `hive.exec.dynamic.partition parameter` value should be **true**. The default value is true.
 
2. Change the dynamic partition mode to strict. In strict mode, at least one partition has to be static. This prevents queries without the partition filter in the WHERE clause. Therefore, it prevents queries that scan all partitions. Navigate to the Hive **Configs** tab, and then set `hive.exec.dynamic.partition.mode` to **strict**. The default value is nonstrict.
 
3. To limit the number of dynamic partitions to be created, modify the ``hive.exec.max.dynamic.partitions` parameter. The default value is 5,000.
 
4. To limit the total number of dynamic partitions per node, modify `hive.exec.max.dynamic.partitions.pernode`. The default value is 2,000.

 
### Enable local mode

Local mode enables Hive to perform all tasks of a job on a single machine, or sometimes in a single process. This improves query performance if the input data is small and the overhead of launching tasks for queries consumes a significant percentage of the overall query execution.

1. To enable local mode, add the` hive.exec.mode.local.auto` parameter to the Custom hive-site panel, as explained earlier.

![Hive exec mode local auto](./media/hdinsight-changing-configs-via-ambari/hive-exec-mode-local-auto.png)


### Set single MapReduce MultiGROUP BY

When this property is set to true, a MultiGROUP BY query with common group by keys will generate a single MapReduce job.

1. To enable this, add the hive.multigroupby.singlereducer parameter to the Custom hive-site pane, as explained earlier.

![Hive set single MapReduce MultiGROUP BY](./media/hdinsight-changing-configs-via-ambari/hive-multigroupby-singlereducer.png)


## Pig optimization

Pig properties can be easily modified from the Ambari web UI to tune Pig queries. Modifying Pig properties from Ambari directly modifies the Pig properties in the `/etc/pig/2.4.2.0-258.0/pig.properties` file.

1. To modify Pig properties, navigate to the Pig **Configs** tab, and then expand the **Advanced pig-properties** pane.

2. Find, uncomment, and change the value of the property you wish to modify.

3. Select **Save** on the top right side of the window to save the new value. Some properties may require a service restart.

![Advanced pig-properties](./media/hdinsight-changing-configs-via-ambari/advanced-pig-properties.png)
 
> Note: The session-level settings override property values in the `pig.properties` file.


### Tune execution engine

Two execution engines are available to execute Pig scripts: MapReduce and Tez. Tez is an optimized engine and is much faster than MapReduce.

1. To modify the execution engine, in the Advanced pig-properties pane, find the property `exectype`.

2. The default value is MapReduce. Change it to **Tez**.


### Enable local mode

Similar to Hive, local mode is used to speed jobs with relatively less amounts of data.

1. To enable the local mode, set `pig.auto.local.enabled` to **true**. The default value is false.

2. Jobs with an input data size less than the `pig.auto.local.input.maxbytes` property value are considered to be small jobs. The default value is 1 GB.


### Copy user jar cache

Pig copies the jar required by UDFs to a distributed cache in order to make them available for task nodes. These jars do not change frequently. If enabled, this setting allows jars to be placed in a cache to reuse them for jobs run by the same user. This results in a minor increase in job performance.

1. To enable, set `pig.user.cache.enabled` to **true**. The default is false.

2. To set the base path of the cached jars, set `pig.user.cache.location` to the base path. The default is /tmp.


### Optimize performance with memory settings

The following memory settings can help optimize Pig script performance.

1. `pig.cachedbag.memusage`: The amount of memory allocated to a bag. A bag is collection of tuples. A tuple is an ordered set of fields, and a field is a piece of data. If the data in a bag is beyond the allocated memory, it is spilled to disk. The default value is 0.2, which represents 20 percent of available memory. This memory is shared across all bags in an application.

2. `pig.spill.size.threshold`: Bags smaller than the spill size threshold (bytes) are not spilled to disk. The default value is 5 MB.


### Compress temporary files

Pig generates temporary files during job execution. Compressing the temporary files results in a performance increase when reading or writing files to disk. The following settings can be used to compress temporary files.

* `pig.tmpfilecompression`: When true, enables temporary file compression. (Default value is false.)

* `pig.tmpfilecompression.codec`: The compression codec to use for compressing the temporary files.

> Note: The recommended compression codecs are LZO and Snappy because of lower CPU utilization.


### Enable split combining

When enabled, small files are combined for fewer map tasks. This improves the efficiency of jobs with many small files. To enable, set `pig.noSplitCombination` to **true**. The default value is false.


### Tune mappers

The number of mappers can be controlled by modifying the property `pig.maxCombinedSplitSize`. This specifies the size of the data to be processed by a single map task. The default value is the filesystems default block size. Increasing this value will result in a decrease of the number of mapper tasks.


### Tune reducers

The number of reducers is calculated based on the parameter `pig.exec.reducers.bytes.per.reducer`. The parameter specifies the number of bytes processed per reducer. The default value is 1 GB. To limit the maximum number of reducers, set the `pig.exec.reducers.max` property. The default value is 999.


## Next steps

* Read more about [managing HDInsight clusters by using the Ambari Web UI](hdinsight-hadoop-manage-ambari)
* Learn how to work with the Ambari [REST API](hdinsight-hadoop-manage-ambari-rest-api)
