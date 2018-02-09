# Lambda-Architecture

Notes on Lambda Architecture

## Why Lambda Architecture

* We are generating data at an unprecedented rate. But data is not the same as knowledge.

* To extract useful insights from the data and to tame the three Vs of data (Volume, Velocity and Variety), we need to rethink our tools and design principles.

* The general transition is as follows:
    * RDBMS
    * Message Queues
    * Sharding

* New set of tools we have:
    * NoSQL Databases - Mongo, Cassandra, HBase
    * Highly Scalable Message Queues - Kafka
    * Distributed filesystems - HDFS
    * MapReduce Paradigm - Hadoop, Spark

* In this series of innovations and improvement, we have an alternate paradigm for Big Data computation - the Lambda Architecture

## Principle

* We want to develop a system that can answer a query about the data. ie **query = function(all data)**

* Effectively, we want to be able to implement arbitrary functions over arbitrary data.

## Building Blocks

* Remember **query = function(data)**. So, in theory, to compute the result, we could run the `function` over the entire data every time.

* But this would be inefficient and expensive.

* So we precompute the query function over the data and store it as a **batch view**.  We can further index these **batch views** to provide query results very fast.

* **batch view = function(all data)**

* **query = function(batch view)**

* Lambda Architecture should have a component to compute the batch views. This component is called as **Batch Layer**.

### Batch Layer

* Batch Layer should be able to do two things:

    * Compute arbitrary functions over arbitrary data ie implement **batch view = function(all data)**

    * Store a copy of immutable, constantly growing, append-only master dataset

* In pseudo code:

    ```
    function batch_layer():
        while (True):
            ingest_data()
            compute_batch_view()
            append_to_master_dataset()

    ```

* Regular-write workload.

* Can be easily scaled horizontally.

* Hadoop is an example of batch processing systems.

* Now we need a component to query over these batch views

### Serving Layer

* Indexes the batch view so that it can be queried efficiently.

* Effectively, the actual query has been broken down into precomputed sub-queries. These sub-queries were executed in the Batch Layer and the results of those sub-queries are not being queried to generate the final result.

* In pseudo code:

    ```
    function serve_layer():
        query_batch_view()
    ```

* When new batches are available, Serving Layer indexes them as well.

* Workload:
    
    * Write at time of batch updates.
    * Random read queries.

* Since no random writes are required, the indexing (and database) layer are very simple.

### Things so far

* We have a two-layer architecture - with Batch and Serving layer.

* We can bake in fault tolerance into both the layers.

* In case there is any human error, the batched views can be simply recomputed from scratch.

* Since Batch Layer can compute arbitrary functions over arbitrary datasets, the architecture is quite general and extensible.

* Further, we can use any combination of technology for the two layers.

* This simple architecture is able to solve a lot of big data use cases.

* The architecture so far is not complete.

* Batch Layer can take a significant amount of time to process a batch of data which makes it a high latency layer.

* While some data is being processed, the new incoming data is queued and is picked as part of the next batch. This means there is always some backlog of data which is not yet consumed.

* One solution: use smaller batches. We could potentially waste a lot of time in context switch and still have a backlog of records (streaming cases). But we will talk about this option a little later.

* Now when we query the Serving Layer, the results do not account for the data which is yet to be processed and this could be a problem depending on the use cases.

* Basically it is a problem for all the streaming/real-time use cases.

* So the next problem we have to solve is how to make the backlog data available for query.

* We introduce one more layer, called as Speed Layer, to solve this problem.

### Speed Layer

* Speed Layer is similar to Batch Layer in terms of how it processes the incoming data but there are some key differences:

    * Speed Layer operates only on the most recent, un-batched data while the Batch Layer operates on the entire data. Further, Speed Layers works incrementally over the data. So if a new record is added to the un-batched data, Speed Layer just processes the newly added record.

    * Batch Layer can support more complex operations than Speed Layer. This is partially because Speed Layer should be low latency and partially because Speed Layer does not have access to entire data. So operations that require past data (eg get me all those current users who visited my site at least once in last one month) cannot be trivially supported by Speed Layer.

    * Batch Layer has a high latency while Speed Layer has a low latency.

    * Speed Layer is a stream processing system while Batch Layer is a, surprise surpise, batch processing system.

* Spark Streaming and Storm are examples of real-time processing systems.

*   ```
    function spped_layer():
        while (True):
            ingest_data()
            compute_speed_view()

    ```

* Speed Layer needs to support both random read and write queries. Hence, it requires a much more complicated database system.

* Once the un-batched data is fed to the Batch Layer and the batch view passed to the Serving Layer, the corresponding results from the Speed Layer are flushed. The complexity of processing the most recent data is pushed to a layer whose results are temporary. This is known as **Complexity Isolation**.

This ensures that every query reads only one copy of each record - either from batch views or from Speed Layer.

* In the case of any failure, the Speed Layer can be reset and all the un-batched data can be recomputed.

* The last piece of the architecture is how to merge the results from Speed Layer and Batch Layer.

## Variants

### Unified Lambda Architecture

* Use the same code for both Batch and Speed Layer and combine their results almost transparently.
* Examples are systems like Apache Spark and Apache Flink.

### Free Lambda Architecture

* Implement each layer independently, maintain them independently and write a system to merge the results depending on the use case/business logic.

## Limitations 

* One should know if they need Lambda Architecture. A large number of big data use cases can be solved using just the Batch Layer.
