---
title: "Async I/O"
weight: 5
type: docs
aliases:
  - /dev/stream/operators/asyncio.html
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# Asynchronous I/O for External Data Access

This page explains the use of Flink's API for asynchronous I/O with external data stores.
For users not familiar with asynchronous or event-driven programming, an article about Futures and
event-driven programming may be useful preparation.

Note: Details about the design and implementation of the asynchronous I/O utility can be found in the proposal and design document
[FLIP-12: Asynchronous I/O Design and Implementation](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=65870673).
Details about the new retry support can be found in document
[FLIP-232: Add Retry Support For Async I/O In DataStream API](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=211883963).

## The need for Asynchronous I/O Operations

When interacting with external systems (for example when enriching stream events with data stored in a database), one needs to take care
that communication delay with the external system does not dominate the streaming application's total work.

Naively accessing data in the external database, for example in a `MapFunction`, typically means **synchronous** interaction:
A request is sent to the database and the `MapFunction` waits until the response has been received. In many cases, this waiting
makes up the vast majority of the function's time.

Asynchronous interaction with the database means that a single parallel function instance can handle many requests concurrently and
receive the responses concurrently. That way, the waiting time can be overlaid with sending other requests and
receiving responses. At the very least, the waiting time is amortized over multiple requests. This leads in most cased to much higher
streaming throughput.

{{< img src="/fig/async_io.svg" width="50%" >}}

*Note:* Improving throughput by just scaling the `MapFunction` to a very high parallelism is in some cases possible as well, but usually
comes at a very high resource cost: Having many more parallel MapFunction instances means more tasks, threads, Flink-internal network
connections, network connections to the database, buffers, and general internal bookkeeping overhead.


## Prerequisites

As illustrated in the section above, implementing proper asynchronous I/O to a database (or key/value store) requires a client
to that database that supports asynchronous requests. Many popular databases offer such a client.

In the absence of such a client, one can try and turn a synchronous client into a limited concurrent client by creating
multiple clients and handling the synchronous calls with a thread pool. However, this approach is usually less
efficient than a proper asynchronous client.


## Async I/O API

Flink's Async I/O API allows users to use asynchronous request clients with data streams. The API handles the integration with
data streams, well as handling order, event time, fault tolerance, retry support, etc.

Assuming one has an asynchronous client for the target database, three parts are needed to implement a stream transformation
with asynchronous I/O against the database:

  - An implementation of `AsyncFunction` that dispatches the requests
  - A *callback* that takes the result of the operation and hands it to the `ResultFuture`
  - Applying the async I/O operation on a DataStream as a transformation with or without retry

The following code example illustrates the basic pattern:

```java
// This example implements the asynchronous request and callback with Futures that have the
// interface of Java 8's futures (which is the same one followed by Flink's Future)

/**
 * An implementation of the 'AsyncFunction' that sends requests and sets the callback.
 */
class AsyncDatabaseRequest extends RichAsyncFunction<String, Tuple2<String, String>> {

    /** The database specific client that can issue concurrent requests with callbacks */
    private transient DatabaseClient client;

    @Override
    public void open(OpenContext openContext) throws Exception {
        client = new DatabaseClient(host, post, credentials);
    }

    @Override
    public void close() throws Exception {
        client.close();
    }

    @Override
    public void asyncInvoke(String key, final ResultFuture<Tuple2<String, String>> resultFuture) throws Exception {

        // issue the asynchronous request, receive a future for result
        final Future<String> result = client.query(key);

        // set the callback to be executed once the request by the client is complete
        // the callback simply forwards the result to the result future
        CompletableFuture.supplyAsync(new Supplier<String>() {

            @Override
            public String get() {
                try {
                    return result.get();
                } catch (InterruptedException | ExecutionException e) {
                    // Normally handled explicitly.
                    return null;
                }
            }
        }).thenAccept( (String dbResult) -> {
            resultFuture.complete(Collections.singleton(new Tuple2<>(key, dbResult)));
        });
    }
}

// create the original stream
DataStream<String> stream = ...;

// apply the async I/O transformation without retry
DataStream<Tuple2<String, String>> resultStream =
    AsyncDataStream.unorderedWait(stream, new AsyncDatabaseRequest(), 1000, TimeUnit.MILLISECONDS, 100);

// or apply the async I/O transformation with retry
// create an async retry strategy via utility class or a user defined strategy
AsyncRetryStrategy asyncRetryStrategy =
	new AsyncRetryStrategies.FixedDelayRetryStrategyBuilder(3, 100L) // maxAttempts=3, fixedDelay=100ms
		.ifResult(RetryPredicates.EMPTY_RESULT_PREDICATE)
		.ifException(RetryPredicates.HAS_EXCEPTION_PREDICATE)
		.build();

// apply the async I/O transformation with retry
DataStream<Tuple2<String, String>> resultStream =
	AsyncDataStream.unorderedWaitWithRetry(stream, new AsyncDatabaseRequest(), 1000, TimeUnit.MILLISECONDS, 100, asyncRetryStrategy);
```

**Important note**: The `ResultFuture` is completed with the first call of `ResultFuture.complete`.
All subsequent `complete` calls will be ignored.

The following three parameters control the asynchronous operations:

  - **Timeout**: The timeout defines the maximum duration from the first invocation to the final completion of an asynchronous operation,
    This duration may include multiple retry attempts (if retries are enabled) and determines when the operation is ultimately considered complete.
    This parameter guards against dead/failed requests.

  - **Capacity**: This parameter defines how many asynchronous requests may be in progress at the same time.
    Even though the async I/O approach leads typically to much better throughput, the operator can still be the bottleneck in
    the streaming application. Limiting the number of concurrent requests ensures that the operator will not
    accumulate an ever-growing backlog of pending requests, but that it will trigger backpressure once the capacity
    is exhausted.

  - **AsyncRetryStrategy**: The asyncRetryStrategy defines what conditions will trigger a delayed retry and the delay strategy,
    e.g., fixed-delay, exponential-backoff-delay, custom implementation, etc.

### Timeout Handling

When an async I/O request times out, by default an exception is thrown and job is restarted.
If you want to handle timeouts, you can override the `AsyncFunction#timeout` method.
Make sure you call `ResultFuture.complete()` or `ResultFuture.completeExceptionally()` when overriding
in order to indicate to Flink that the processing of this input record has completed. You can call 
`ResultFuture.complete(Collections.emptyList())` if you do not want to emit any record when timeouts happen.


### Order of Results

The concurrent requests issued by the `AsyncFunction` frequently complete in some undefined order, based on which request finished first.
To control in which order the resulting records are emitted, Flink offers two modes:

  - **Unordered**: Result records are emitted as soon as the asynchronous request finishes.
    The order of the records in the stream is different after the async I/O operator than before.
    This mode has the lowest latency and lowest overhead, when used with *processing time* as the basic time characteristic.
    Use `AsyncDataStream.unorderedWait(...)` for this mode.

  - **Ordered**: In that case, the stream order is preserved. Result records are emitted in the same order as the asynchronous
    requests are triggered (the order of the operators input records). To achieve that, the operator buffers a result record
    until all its preceding records are emitted (or timed out).
    This usually introduces some amount of extra latency and some overhead in checkpointing, because records or results are maintained
    in the checkpointed state for a longer time, compared to the unordered mode.
    Use `AsyncDataStream.orderedWait(...)` for this mode.


### Event Time

When the streaming application works with [event time]({{< ref "docs/concepts/time" >}}), watermarks will be handled correctly by the
asynchronous I/O operator. That means concretely the following for the two order modes:

  - **Unordered**: Watermarks do not overtake records and vice versa, meaning watermarks establish an *order boundary*.
    Records are emitted unordered only between watermarks.
    A record occurring after a certain watermark will be emitted only after that watermark was emitted.
    The watermark in turn will be emitted only after all result records from inputs before that watermark were emitted.

    That means that in the presence of watermarks, the *unordered* mode introduces some of the same latency and management
    overhead as the *ordered* mode does. The amount of that overhead depends on the watermark frequency.

  - **Ordered**: Order of watermarks and records is preserved, just like order between records is preserved. There is no
    significant change in overhead, compared to working with *processing time*.

Please recall that *Ingestion Time* is a special case of *event time* with automatically generated watermarks that
are based on the sources processing time.


### Fault Tolerance Guarantees

The asynchronous I/O operator offers full exactly-once fault tolerance guarantees. It stores the records for in-flight
asynchronous requests in checkpoints and restores/re-triggers the requests when recovering from a failure.


### Retry Support

The retry support introduces a built-in mechanism for async operator which being transparently to the user's AsyncFunction.

  - **AsyncRetryStrategy**: The `AsyncRetryStrategy` contains the definition of the retry condition `AsyncRetryPredicate` and the interfaces
    to determine whether to continue retry and the retry interval based on the current attempt number.
    Note that after the trigger retry condition is met, it is possible to abandon the retry because the current attempt number exceeds the preset limit,
    or to be forced to terminate the retry at the end of the task (in this case, the system takes the last execution result or exception as the final state).

  - **AsyncRetryPredicate**: The retry condition can be triggered based on the return result or the execution exception.


### Implementation Tips

For implementations with *Futures* that have an *Executor* for callbacks, we suggest using a `DirectExecutor`, because the
callback typically does minimal work, and a `DirectExecutor` avoids an additional thread-to-thread handover overhead. The callback typically only hands
the result to the `ResultFuture`, which adds it to the output buffer. From there, the heavy logic that includes record emission and interaction
with the checkpoint bookkeeping happens in a dedicated thread-pool anyways.

A `DirectExecutor` can be obtained via `org.apache.flink.util.concurrent.Executors.directExecutor()` or
`com.google.common.util.concurrent.MoreExecutors.directExecutor()`.


### Caveats

**The AsyncFunction is not called Multi-Threaded**

A common confusion that we want to explicitly point out here is that the `AsyncFunction` is not called in a multi-threaded fashion.
There exists only one instance of the `AsyncFunction` and it is called sequentially for each record in the respective partition
of the stream. Unless the `asyncInvoke(...)` method returns fast and relies on a callback (by the client), it will not result in
proper asynchronous I/O.

For example, the following patterns result in a blocking `asyncInvoke(...)` functions and thus void the asynchronous behavior:

  - Using a database client whose lookup/query method call blocks until the result has been received back

  - Blocking/waiting on the future-type objects returned by an asynchronous client inside the `asyncInvoke(...)` method
  
**An AsyncFunction(AsyncWaitOperator) can be used anywhere in the job graph, except that it cannot be chained to a `SourceFunction`/`SourceStreamTask`.**

**May Need Larger Queue Capacity If Retry Enabled**

The new retry feature may result in larger queue capacity requirements, the maximum number can be approximately evaluated as below:

```
inputRate * retryRate * avgRetryDuration
```

For example, for a task with inputRate = 100 records/sec, where 1% of the elements will trigger 1 retry on average, and the average retry time is 60s,
the additional queue capacity requirement will be:

```
100 records/sec * 1% * 60s = 60
```

That is, adding more 60 capacity to the work queue may not affect the throughput in unordered output mode , in case of ordered mode, the head element is the key point,
and the longer it stays uncompleted, the longer the processing delay provided by the operator, the retry feature may increase the incomplete time of the head element,
if in fact more retries are obtained with the same timeout constraint.

When the queue capacity grows(common way to ease the backpressure), the risk of OOM increases. Though in fact, for `ListState` storage, the theoretical upper limit is `Integer.MAX_VALUE`,
so the queue capacity's limit is the same, but we can't increase the queue capacity too big in production, increase the task parallelism maybe a more viable way.

{{< top >}}
