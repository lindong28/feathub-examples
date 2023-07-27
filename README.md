# Summary

## FLIP-331: Use blocking ResultPartitionType if operator only outputs records on EOF

FLIP link: https://cwiki.apache.org/confluence/display/FLINK/FLIP-331%3A+Use+blocking+ResultPartitionType+if+operator+only+outputs+records+on+EOF

PR link: https://github.com/apache/flink/pull/23000

### Motivation

 ```
 	As shown in the figure below, we might have a job that pre-processes records from a bounded source (i.e. inputA) using an operator (i.e. operatorA) which only emits results after its input has ended. Then operatorA needs to join records emitted by operatorA with records from an unbounded source, and emit results with low processing latency in real-time.
   Currently, supporting the above use-case requires all operators to be deployed at the start of the job. This approach wastes slot resources because operatorB can not do any useful work until operatorA's input has ended. Even worse, operatorB might use a lot of disk space only to cache and spill records received from the unbounded source to its local disk while it is waiting for operatorA's output.
 	In this FLIP, we propose to optimize performance for the above use-case by allowing an operator to explicitly specify whether it only emits records after all its inputs have ended. JM will leverage this information to optimize job scheduling such that the partition type of the results emitted by this operator, as well as the results emitted by its upstream operators, will all be blocking, which effectively let Flink schedule and execute this operator as well as its upstream operators in batch mode.
 ```

![1](https://p.ipic.vip/d4nx89.png)

### Target

Block operatorA's output Edge, in which case operatorB will not be deployed before operatorA has finished its work.



### First Try

- Configure operatorA transformation to make it a ExecutionMode-BATCH-like operator: keyed transformation require to sort before.

- Change the ExchangeMode of the outputOnEOF transformation to BATCH when build the StreamGraph.

- After above changes, operatorA did not get a better performance. By reading code, set BatchStateBackendAndTimerService for streaming work to avoid scan each element of a sorted stream of records. Finally, operatorA in STREAMING gets a close performance as in BATCH mode.

- Problem Summary:

   1). Blocking edge in StreamGraph influences operator chain.

   ```
   if ((partitioner instanceof ForwardPartitioner)
           && exchangeMode != StreamExchangeMode.BATCH)
   ```

   2). The upstream checkpoint can not take effect because of the Blocking Edge.

   3). Specific operator with an internal sorter such as newly added `EOFCoGroupOperator` may cause repeating sort problem.



### Final Change

- Add `OperatorAttributesBuilder` and `OperatorAttributes` for operator developers to specify operator attributes that Flink runtime can use to optimize the job performance.
- Add internal method to `StreamOperator` to allow implementations to report whether the operator will output records on end of input and whether it has a internal sort.
- Optimize job when schedule and execute the output-on-EOF operator as well as its upstream operators in batch mode.
- Set the sort requirement and memory use case weights for the keyed input stream when translating transformation to take advantage of outputOnEOF.
- Extra work
  - Fix an unstable test `SinkTransformationTranslatorITCase` in which `findCommitter` may get the node "Global Committer" instead of "Committer".
  - Add the `EOFCoGroupOperator` developed by Lin Dong, and apply it when the window assigner is `EndOfStreamWindow ` in which case, DataStream#coGroup in streaming mode can be optimized 20X faster.



### Benchmark

​	We use a user-defined function which just invokes "collector.collect(1)" and both operators do none of specific transformations so that overhead of the user-defined function is minimized.We run the  benchmark on a mackbook with the latest Flink 1.18-snapshot and parallelism=1. RocksDB is used in the streaming mode.

1) Use DataStream#coGroup to process records from two bounded streams and emit results after both inputs have ended.

   [CoGroupDataStream](https://github.com/lindong28/flink/blob/optimize-cogroup/flink-examples/flink-examples-streaming/src/main/java/org/apache/flink/streaming/examples/CoGroupDataStream.java)

   ```java
   data1.coGroup(data2)
        .where(tuple -> tuple.f0)
        .equalTo(tuple -> tuple.f0)
        .window(EndOfStreamWindows.get())
        .apply(new CustomCoGroupFunction())
        .addSink(...);
   ```

   Here are the benchmark results:

   - Without the proposed change, in stream mode, with each of sources generating 2*10^6 records, average execution time is 56 sec.
   - Without the proposed change, in batch mode, with each of sources generating 5*10^7 records, average execution time is 118 sec.
   - With the proposed change, in both the stream and batch mode, with each of sources generating 5*10^7 records, average execution time is 46 sec.
   
   This shows with the changes proposed above, DataStream#coGroup can be 20X faster than its current throughput in stream mode, and 2.5X faster than its current throughput in batch mode.

2. We have a job that pre-processes records from a bounded source (inputA) using an operator (operatorA) which only emits results after its input has ended. Then operatorA needs to process records emitted by operatorA with records from an unbounded source, and emit results with low processing latency in real-time.

   Even though batch mode can not meet needs in this scene, we get its benchmark as a reference.

   [OperatorDelayedDeployDataStream](https://github.com/apache/flink/blob/e248d36fbe260acea314452b2b7d2994ffa8a034/flink-examples/flink-examples-streaming/src/main/java/org/apache/flink/streaming/examples/ml/OperatorDelayedDeployDataStream.java)
   
   ```java
   sourceStream.transform(
               "Process1",
               Types.TUPLE(Types.INT, Types.INT, Types.DOUBLE),
               new TestOutputEOFProcessOperator<>(env.clean(processFunction)))
               .connect(unboundedSourceSteam)
               .transform("Process2", Types.INT, new MyProcessOperator())
               .addSink(...);
   ```
   
   Here are the benchmark results:
   
   - Without the proposed change, in stream mode, with each of sources generating 3*10^7 records, average execution time is 50 sec.
   - Without the proposed change, in batch mode, with each of sources generating 3*10^8 records, average execution time is 72 sec.
   - With the proposed change, in stream mode, with each of sources generating 3*10^8 records, average execution time is 51 sec.
   
   This shows that,  with the changes proposed above, the total processing time can be about 10X faster than its current processing time in this scene. As the records number grows, more memory will be used, and this performance gap will increase continuously.