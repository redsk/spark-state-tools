# Spark State Tools 

[![CircleCI](https://circleci.com/gh/redsk/spark-state-tools/tree/master.svg?style=svg)](https://circleci.com/gh/redsk/spark-state-tools/tree/master)

Spark State Tools provides features about offline manipulation of Structured Streaming state on existing query.

The features we provide as of now are:

* Show some state information which you'll need to provide to enjoy below features
  * state operator information from checkpoint
  * state schema from streaming query
* Create savepoint from existing checkpoint of Structured Streaming query
  * You can pick specific batch (if it exists on metadata) to create savepoint
* Read state as batch source of Spark SQL
* Write DataFrame to state as batch sink of Spark SQL
  * With feature of writing state, you can achieve rescaling state (repartition), simple schema evolution, etc.
* Migrate state format from old to new
  * migrating Streaming Aggregation from ver 1 to 2
  * migrating FlatMapGroupsWithState from ver 1 to 2

As this project leverages Spark Structured Streaming's interfaces, and doesn't deal with internal
(e.g. the structure of state file for HDFS state store), the performance may be suboptimal.

For now, from the most parts, states from Streaming Aggregation query (`groupBy().agg()`) and (Flat)MapGroupsWithState are supported.

## Disclaimer

This is something more of a proof of concept implementation, might not be something for production ready.
When you deal with writing state, you may want to backup your checkpoint with CheckpointUtil and try doing it with savepoint.

The project is intended to deal with offline state, not against state which streaming query is running.
Actually it can be possible, but state store provider in running query can purge old batches, which would produce error on here.

## Supported versions

Spark 2.4.x is supported: it only means you should link Spark 2.4.x when using this tool. That state formats across the Spark 2.x versions are supported.

The project doesn't support cross-scala versions: Scala 2.11.x is supported only.

## Pulling artifacts

You may use this library in your applications with the following dependency information:

```
groupId: net.heartsavior.spark
artifactId: spark-state-tools
```

You are encouraged to always use latest version which is compatible to your Apache Spark version.

e.g. For maven:

```
<dependency>
  <groupId>net.heartsavior.spark</groupId>
  <artifactId>spark-state-tools</artifactId>
  <version>0.2.0</version>
</dependency>
```

For other dependency managements, you can refer below page to get the guide:

https://search.maven.org/artifact/net.heartsavior.spark/spark-state-tools/


## How to use

First of all, you may want to get state and last batch information to provide them as parameters.
You can get it from `StateInformationInCheckpoint`, whether calling from your codebase or running with `spark-submit`.
Here we assume you have artifact jar of spark-state-tool and you want to run it from cli (leveraging `spark-submit`).

```text
<spark_path>/bin/spark-submit --master "local[*]" \
--class net.heartsavior.spark.sql.state.StateInformationInCheckpoint \
spark-state-tool-0.0.1-SNAPSHOT.jar <checkpoint_root_path>
```

The command line will provide checkpoint information like below:

```text
Last committed batch ID: 2
Operator ID: 0, partitions: 5, storeNames: List(default)
```

This output means the query has batch ID 2 as last committed (NOTE: corresponding state version is 3, not 2), and
there's only one stateful operator which has ID as 0, and 5 partitions, and there's also only one kind of store named "default".

You can achieve this as calling `StateInformationInCheckpoint.gatherInformation` against checkpoint directory too.

```scala
// Here we assume 'spark' as SparkSession.
// Here the class of Path is `org.apache.hadoop.fs.Path`
val stateInfo = new StateInformationInCheckpoint(spark).gatherInformation(new Path(cpDir.getAbsolutePath))
// Here stateInfo is `StateInformation`, which you can extract same information as running CLI app
```

To read state from your existing query, you may want to provide state schema manually, or read from your existing query:

* Read schema from existing query

(supported: `streaming aggregation`, `flatMapGroupsWithState`)

```scala
// Here we assume 'spark' as SparkSession.
// the query shouldn't have sink - you may need to get rid of writeStream part and pass DataFrame
val schemaInfos = new StateSchemaExtractor(spark).extract(streamingQueryDf)
// Here schemaInfos is `Seq[StateSchemaInfo]`, which you can extract keySchema,
// and valueSchema and finally define state schema. Please refer "Manual schema"
// to define state schema with key schema and value schema
```

* Manual schema

```scala
val stateKeySchema = new StructType()
  .add("groupKey", IntegerType)

val stateValueSchema = new StructType()
  .add("cnt", LongType)
  .add("sum", LongType)
  .add("max", IntegerType)
  .add("min", IntegerType)

val stateFormat = new StructType()
  .add("key", stateKeySchema)
  .add("value", stateValueSchema)
```

You can also combine both state operator information in state information and state schema via `StateStoreReaderOperatorParamExtractor`
to get necessary parameters for state batch read:

```scala
// Here we assume 'spark' as SparkSession.
val stateInfo = new StateInformationInCheckpoint(spark).gatherInformation(new Path(cpDir.getAbsolutePath))
val schemaInfos = new StateSchemaExtractor(spark).extract(streamingQueryDf)
val stateReadParams = StateStoreReaderOperatorParamExtractor.extract(stateInfo, schemaInfos)
// from `stateReadParams` you can get last committed state version, operatorId, storeName, state schema per each (operatorId, storeName) group
```

Then you can start your batch query like:

```scala
val operatorId = 0
val batchId = 1 // the version of state for the output of batch is batchId + 1

// Here we assume 'spark' as SparkSession
val stateReadDf = spark.read
  .format("state")
  .schema(stateSchema)
  .option(StateStoreDataSourceProvider.PARAM_CHECKPOINT_LOCATION,
    new Path(checkpointRoot, "state").getAbsolutePath)
  .option(StateStoreDataSourceProvider.PARAM_VERSION, batchId + 1)
  .option(StateStoreDataSourceProvider.PARAM_OPERATOR_ID, operatorId)
  .load()


// The schema of stateReadDf follows:
// For streaming aggregation state format v1
// (query ran with lower than Spark 2.4.0 for the first time)
/*
root
 |-- key: struct (nullable = false)
 |    |-- groupKey: integer (nullable = true)
 |-- value: struct (nullable = false)
 |    |-- groupKey: integer (nullable = true)
 |    |-- cnt: long (nullable = true)
 |    |-- sum: long (nullable = true)
 |    |-- max: integer (nullable = true)
 |    |-- min: integer (nullable = true)
*/

// For streaming aggregation state format v2
// (query ran with Spark 2.4.0 or higher for the first time)
/*
root
 |-- key: struct (nullable = false)
 |    |-- groupKey: integer (nullable = true)
 |-- value: struct (nullable = false)
 |    |-- cnt: long (nullable = true)
 |    |-- sum: long (nullable = true)
 |    |-- max: integer (nullable = true)
 |    |-- min: integer (nullable = true)
*/
```

To write Dataset as state of Structured Streaming, you can transform your Dataset as having schema as follows:

```text
root
 |-- key: struct (nullable = false)
 |    |-- ...key fields...
 |-- value: struct (nullable = false)
 |    |-- ...value fields...
```

and add state batch output as follow:

```scala
val operatorId = 0
val batchId = 1 // the version of state for the output of batch is batchId + 1
val newShufflePartitions = 10

df.write
  .format("state")
  .option(StateStoreDataSourceProvider.PARAM_CHECKPOINT_LOCATION,
    new Path(newCheckpointRoot, "state").getAbsolutePath)
  .option(StateStoreDataSourceProvider.PARAM_VERSION, batchId + 1)
  .option(StateStoreDataSourceProvider.PARAM_OPERATOR_ID, operatorId)
  .option(StateStoreDataSourceProvider.PARAM_NEW_PARTITIONS, newShufflePartitions)
  .save() // saveAsTable() also supported
```

Before that, you may want to create a savepoint from existing checkpoint to another path, so that you can simply 
run new Structured Streaming query with modified state.

```scala
// Here we assume 'spark' as SparkSession.
// If you just want to create a savepoint without modifying state, provide `additionalMetadataConf` as `Map.empty`,
// and `excludeState` as `false`.
// That said, if you want to prepare state modification, it would be good to create a savepoint with providing
// addConf to new shuffle partition (like below), and `excludeState` as `true` (to avoid unnecessary copy for state)
val addConf = Map(SQLConf.SHUFFLE_PARTITIONS.key -> newShufflePartitions.toString)
CheckpointUtil.createSavePoint(spark, oldCpPath, newCpPath, newLastBatchId, addConf, excludeState = true)
```

If you ran streaming aggregation query before Spark 2.4.0 and want to upgrade (or already upgraded) to Spark 2.4.0 or higher,
you may also want to migrate your state from state format 1 to 2 (Spark 2.4.0 introduces it) to reduce overall state size,
and get some speedup from most of cases.

Please refer [SPARK-24763](https://issues.apache.org/jira/browse/SPARK-24763) for more details.

```scala
// Here we assume 'spark' as SparkSession.

// Please refer above to see how to construct `stateSchema`
// (manually, or reading from existing query)
// Here we already construct `stateSchema` as state schema.

val migrator = new StreamingAggregationMigrator(spark)
migrator.convertVersion1To2(oldCpPath, newCpPath, stateKeySchema, stateValueSchema)
```

Similarly, if you ran flatMapGroupsWithState query before Spark 2.4.0 and want to upgrade (or already upgraded) to Spark 2.4.0 or higher,
you may also want to migrate your state from state format 1 to 2 (Spark 2.4.0 introduces it) to enable setting timeout even when state is null.
(This also changes timeout timestamp type from int to long.)

Please refer [SPARK-22187](https://issues.apache.org/jira/browse/SPARK-22187) for more details.

```scala
// Here we assume 'spark' as SparkSession.

// Please refer above to see how to construct `stateSchema`
// (manually, or reading from existing query)
// Here we already construct `stateSchema` as state schema.

val migrator = new FlatMapGroupsWithStateMigrator(spark)
migrator.convertVersion1To2(oldCpPath, newCpPath, stateKeySchema, stateValueSchema)
```

Please refer the [test codes](https://github.com/HeartSaVioR/spark-state-tools/tree/master/src/test/scala/net/heartsavior/spark/sql/state) to see more examples on how to use.

## License

Copyright 2019 Jungtaek Lim "<kabhwan@gmail.com>"

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License. 
