# Chapter 2
## WordCount的一生
笔者将以flink/flink-examples/flink-examples-streaming/src/main/java/org/apache/flink/streamin/examples/wordcount/WordCount.java为例子简单说明其运行过程。源码如下所示：  
```java
    /*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *    http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.apache.flink.streaming.examples.wordcount;

import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.common.serialization.SimpleStringEncoder;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.configuration.MemorySize;
import org.apache.flink.connector.file.sink.FileSink;
import org.apache.flink.connector.file.src.FileSource;
import org.apache.flink.connector.file.src.reader.TextLineInputFormat;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.sink.filesystem.rollingpolicies.DefaultRollingPolicy;
import org.apache.flink.streaming.examples.wordcount.util.CLI;
import org.apache.flink.streaming.examples.wordcount.util.WordCountData;
import org.apache.flink.util.Collector;

import java.time.Duration;

/**
 * Implements the "WordCount" program that computes a simple word occurrence histogram over text
 * files. This Job can be executed in both streaming and batch execution modes.
 *
 * <p>The input is a [list of] plain text file[s] with lines separated by a newline character.
 *
 * <p>Usage:
 *
 * <ul>
 *   <li><code>--input &lt;path&gt;</code>A list of input files and / or directories to read. If no
 *       input is provided, the program is run with default data from {@link WordCountData}.
 *   <li><code>--discovery-interval &lt;duration&gt;</code>Turns the file reader into a continuous
 *       source that will monitor the provided input directories every interval and read any new
 *       files.
 *   <li><code>--output &lt;path&gt;</code>The output directory where the Job will write the
 *       results. If no output path is provided, the Job will print the results to <code>stdout
 *       </code>.
 *   <li><code>--execution-mode &lt;mode&gt;</code>The execution mode (BATCH, STREAMING, or
 *       AUTOMATIC) of this pipeline.
 * </ul>
 *
 * <p>This example shows how to:
 *
 * <ul>
 *   <li>Write a simple Flink DataStream program
 *   <li>Use tuple data types
 *   <li>Write and use a user-defined function
 * </ul>
 */
public class WordCount {

    // *************************************************************************
    // PROGRAM
    // *************************************************************************

    public static void main(String[] args) throws Exception {
        final CLI params = CLI.fromArgs(args);

        // Create the execution environment. This is the main entrypoint
        // to building a Flink application.
        final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // Apache Flink’s unified approach to stream and batch processing means that a DataStream
        // application executed over bounded input will produce the same final results regardless
        // of the configured execution mode. It is important to note what final means here: a job
        // executing in STREAMING mode might produce incremental updates (think upserts in
        // a database) while in BATCH mode, it would only produce one final result at the end. The
        // final result will be the same if interpreted correctly, but getting there can be
        // different.
        //
        // The “classic” execution behavior of the DataStream API is called STREAMING execution
        // mode. Applications should use streaming execution for unbounded jobs that require
        // continuous incremental processing and are expected to stay online indefinitely.
        //
        // By enabling BATCH execution, we allow Flink to apply additional optimizations that we
        // can only do when we know that our input is bounded. For example, different
        // join/aggregation strategies can be used, in addition to a different shuffle
        // implementation that allows more efficient task scheduling and failure recovery behavior.
        //
        // By setting the runtime mode to AUTOMATIC, Flink will choose BATCH if all sources
        // are bounded and otherwise STREAMING.
        env.setRuntimeMode(params.getExecutionMode());

        // This optional step makes the input parameters
        // available in the Flink UI.
        env.getConfig().setGlobalJobParameters(params);

        DataStream<String> text;
        if (params.getInputs().isPresent()) {
            // Create a new file source that will read files from a given set of directories.
            // Each file will be processed as plain text and split based on newlines.
            FileSource.FileSourceBuilder<String> builder =
                    FileSource.forRecordStreamFormat(
                            new TextLineInputFormat(), params.getInputs().get());

            // If a discovery interval is provided, the source will
            // continuously watch the given directories for new files.
            params.getDiscoveryInterval().ifPresent(builder::monitorContinuously);

            text = env.fromSource(builder.build(), WatermarkStrategy.noWatermarks(), "file-input");
        } else {
            text = env.fromElements(WordCountData.WORDS).name("in-memory-input");
        }

        DataStream<Tuple2<String, Integer>> counts =
                // The text lines read from the source are split into words
                // using a user-defined function. The tokenizer, implemented below,
                // will output each word as a (2-tuple) containing (word, 1)
                text.flatMap(new Tokenizer())
                        .name("tokenizer")
                        // keyBy groups tuples based on the "0" field, the word.
                        // Using a keyBy allows performing aggregations and other
                        // stateful transformations over data on a per-key basis.
                        // This is similar to a GROUP BY clause in a SQL query.
                        .keyBy(value -> value.f0)
                        // For each key, we perform a simple sum of the "1" field, the count.
                        // If the input data stream is bounded, sum will output a final count for
                        // each word. If it is unbounded, it will continuously output updates
                        // each time it sees a new instance of each word in the stream.
                        .sum(1)
                        .name("counter");

        if (params.getOutput().isPresent()) {
            // Given an output directory, Flink will write the results to a file
            // using a simple string encoding. In a production environment, this might
            // be something more structured like CSV, Avro, JSON, or Parquet.
            counts.sinkTo(
                            FileSink.<Tuple2<String, Integer>>forRowFormat(
                                            params.getOutput().get(), new SimpleStringEncoder<>())
                                    .withRollingPolicy(
                                            DefaultRollingPolicy.builder()
                                                    .withMaxPartSize(MemorySize.ofMebiBytes(1))
                                                    .withRolloverInterval(Duration.ofSeconds(10))
                                                    .build())
                                    .build())
                    .name("file-sink");
        } else {
            counts.print().name("print-sink");
        }

        // Apache Flink applications are composed lazily. Calling execute
        // submits the Job and begins processing.
        env.execute("WordCount");
    }

    // *************************************************************************
    // USER FUNCTIONS
    // *************************************************************************

    /**
     * Implements the string tokenizer that splits sentences into words as a user-defined
     * FlatMapFunction. The function takes a line (String) and splits it into multiple pairs in the
     * form of "(word,1)" ({@code Tuple2<String, Integer>}).
     */
    public static final class Tokenizer
            implements FlatMapFunction<String, Tuple2<String, Integer>> {

        @Override
        public void flatMap(String value, Collector<Tuple2<String, Integer>> out) {
            // normalize and split the line
            String[] tokens = value.toLowerCase().split("\\W+");

            // emit the pairs
            for (String token : tokens) {
                if (token.length() > 0) {
                    out.collect(new Tuple2<>(token, 1));
                }
            }
        }
    }
}
```
### 准备环境
当我们想利用flink运行这个WordCount.java前，我们需要先准备一个执行环境，否则当我们提交该作业时，因为不知提交到哪里，而导致进程阻塞。可以运行一下命令，以单机模式在本地部署Flink：  

    $ ./bin/start-cluster.sh
    Starting cluster.
    Starting standalonesession daemon on host LAPTOP-XIAOXIN-KONJAC.
    Starting taskexecutor daemon on host LAPTOP-XIAOXIN-KONJAC.
在start-cluster.sh中依次运行两个类：  
1、org.apache.flink.runtime.entrypoint.StandaloneSessionClusterEntrypoint  
该类在flink仓库中的flink-runtime/src/main/java/org/apache/flink/runtime/entrypoint/StandaloneSessionClusterEntrypoint.java路径下  
```java
    /** Entry point for the standalone session cluster. */
public class StandaloneSessionClusterEntrypoint extends SessionClusterEntrypoint {

    public StandaloneSessionClusterEntrypoint(Configuration configuration) {
        super(configuration);
    }

    @Override
    protected DefaultDispatcherResourceManagerComponentFactory
            createDispatcherResourceManagerComponentFactory(Configuration configuration) {
        return DefaultDispatcherResourceManagerComponentFactory.createSessionComponentFactory(
                StandaloneResourceManagerFactory.getInstance());
    }
    public static void main(String[] args) {
        // startup checks and logging
        EnvironmentInformation.logEnvironmentInfo(
                LOG, StandaloneSessionClusterEntrypoint.class.getSimpleName(), args);
        SignalHandler.register(LOG);
        JvmShutdownSafeguard.installAsShutdownHook(LOG);

        final EntrypointClusterConfiguration entrypointClusterConfiguration =
                ClusterEntrypointUtils.parseParametersOrExit(
                        args,
                        new EntrypointClusterConfigurationParserFactory(),
                        StandaloneSessionClusterEntrypoint.class);
        Configuration configuration = loadConfiguration(entrypointClusterConfiguration);

        StandaloneSessionClusterEntrypoint entrypoint =
                new StandaloneSessionClusterEntrypoint(configuration);

        ClusterEntrypoint.runClusterEntrypoint(entrypoint);
    }
}
```
它的作用是启动包含JobManager和TaskManager的Flink集群。  

2、org.apache.flink.runtime.taskexecutor.TaskManagerRunner  
该类在flink仓库中的flink-runtime/src/main/java/org/apache/flink/runtime/taskexecutor/TaskManagerRunner.java路径下，由于代码量较大，这里就不再粘贴，需要可以自行查看。它的作用是启动TaskManager。  

### 提交作业
运行如下命令，可以提交WordCount作业：  

    $ ./bin/flink run examples/streaming/WordCount.jar

#### 程序入口
输入上述命令后，会运行org.apache.flink.client.cli.CliFrontend中的main方法  
该类在flink仓库中的flink-clients/src/main/java/org/apache/flink/client/cli/CliFrontend.java路径下，其作用是为用户提供了一个交互式的方式来管理和提交Flink作业。  

#### 解析参数
解析参数，选择Default命令行接口，然后调用命令行接口继续运行。命令行接口会调用`CliFrontendParser`解析参数，打包有效配置，创建`PackagedProgram`。

#### 调用作业的main方法
调用`ClientUtils`运行程序，设置执行环境的上下文，然后执行作业WordCount的main方法。  

#### 调用执行环境的execute方法  
WordCount会运行如下代码获取一个流执行环境
```java
    final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
```
然后调用其execute方法。  

#### 生成jobGraph和clusterClient
流执行环境会调用`getStreamGraph`得到`streamGraph`，选择并创建`PipelineExecutor`。  
```java
    public StreamGraph(
            ExecutionConfig executionConfig,
            CheckpointConfig checkpointConfig,
            SavepointRestoreSettings savepointRestoreSettings) {
        this.executionConfig = checkNotNull(executionConfig);
        this.checkpointConfig = checkNotNull(checkpointConfig);
        this.savepointRestoreSettings = checkNotNull(savepointRestoreSettings);

        // create an empty new stream graph.
        clear();
    }
```
`PipelineExecutor`调用`PipelineExecutorUtils`的 `getJobGraph`方法得到`jobGraph`。然后通过工厂模式依次生成`clusterDescriptor`、`clusterClientProvider`、`clusterClient`。  

#### 提交作业并返回结果
`clusterClient`提交任务到集群，并且返回提交结果，随后作业WordCount在集群上开始运行。
```java
    CompletableFuture<JobID> submitJob(JobGraph jobGraph)
```