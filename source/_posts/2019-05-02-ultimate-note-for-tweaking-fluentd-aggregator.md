---
title: Performance tweaking for fluentd aggregator (EFK stack)
catalog: true
date: 2019-05-02 18:03:09
subtitle:
header-img: ./fluent.jpeg
tags:
  - fluentd
  - elasticsearch
---

## Preface

Logging is one of the critical components for developers. Every time when things went wrong, we had no doubt but checked what's going on in logs. `Fluentd` is an open source data collector solution which provides many input/output plugins to help us organize our logging layer. There are tons of articles describing the benefits of using `Fluentd` such as buffering, retries and error handling. In this note I don't plan to describe it again, instead, I will address more how to tweak the performance of `Fluentd` aggregator. Especially the case I use the most when fluentd talks to elasticsearch.

The typical way to utilize fluentd is like the following architecture. We can use sidecar fluentd container to collect application logs and transfer logs to fluentd aggregator. By adopting sidecar pattern, fluentd will take care of error handling to deal with network transient failures. Moreover, our application can write logs asynchronously to fluentd sidecar which prevents our application from being affected once remote logging system becomes unstable.

![img](https://docs.fluentd.org/images/fluentd_ha.png)

To understand more benefits, I suggest you guys take a look at this [youtube video](https://www.youtube.com/watch?v=aeGADcC-hUA) which gives a really great explanation. 

## Problem

Since many fluentd sidecars write their logs to fluentd aggregator, soon or later you will face some performance issues. For example, if our aggregator attempts to write logs to elasticsearch, but the write compacity of elasticsearch is insufficient. Then you will see a lot of 503 returns from elasticsearch and fluetnd aggregator has no other choices but keep records in the local buffer (in memory or files). The worst scenario is we run out of the buffer space and start dropping our records. There are 2 possible solutions comes to my mind to tackle this situation:

1. Increase the size of elasticsearch, though it's easy for me to change elasticsearch size (yes, I use AWS managed elasticsearch), this makes us spend more money on elasticserach nodes.
2. Tweak the fluentd aggregator parameters to see if we can improve the bottleneck.

So before I increase elasticsearch node size, I tend to try option 2 to see how much performance can be improved by tuning the parameters.

##  Understand Buffer plugin

![400x400](./architecture.png)

This picture borrowed from this [official slides](https://www.slideshare.net/tagomoris/fluentd-overview-now-and-then). Let's see how fluentd work internally. Here we only focus on input & buffer 

### Input 

When messages come in, it would be assigned a `timestamp` and a `tag`. Messages itself is wrapped as a`record` which is structured JSON format. `timestamp` + `tag` + `record` is called `event`.

```
Timestamp: 2019-05-04 01:22:12
Tag: app.production
Record: {
	"path": "/api/test",
	"user": "john"
}
```



### Buffer

![400x400](https://docs.fluentd.org/images/fluentd-v0.14-plugin-api-overview.png)

According to the document of fluentd, buffer is essentially a set of chunk.  Chunk is filled by incoming `events` and is written into file or memory. Buffer actually has 2 stages to store chunks. These 2 stages are called `stage` and `queue` respectively. Typically buffer has an `enqueue thread` which pushes chunks to queue. Buffer also has a `flush thread` to write chunks to destination.

#### Stage

`chunk` is allocated and filled in the `stage` level. Here we can specify some parameters to change the behavior of allocation and flushing.

1. `chunk_limit_size` decides max size of each chunks
2. `chunk_limit_records` the max number of events that each chunks have
3. `flush_interval` defines how often it invokes `enqueue`, this only works when `flush_mode` being set to `interval`

The `enqueue thread` will write chunk to queue based on the size and flush interval so that we can decide if we care more about latency or throughput (send more data or send data more frequent).

#### queue

`queue` stores chunks and `flush thread` dequeues chunk from queue.

1. `flush_thread_count`: we can launch more than 1 `flush thread`, which can help us flush chunk in parallel.
2. `flush_thread_interval` define interval to invoke flush thread
3. `flush_thread_burst_interval` if buffer queue is nearly full, how often flush thread will be invoked.

Typically we will increase `flush_thread_count` to increase throughput and also deal with network transient failure. see https://github.com/uken/fluent-plugin-elasticsearch#suggested-to-increase-flush_thread_count-why

#### Other parameters

- `total_limit_size` total buffer size (chunk size + queue size)
- `overflow_action` when buffer is full, what kind of action we need to take

#### Note

Buffer plugin is extremely useful when the output destination provides bulk or batch API. So that we are able to flush whole `chunk` content at once by using those APIs instead of sending request multiple times. It's the secret why many fluetnd output plugins make use of buffer plugins. For understanding the further detail, I suggest you guys go through the [source code](https://github.com/fluent/fluentd/blob/master/lib/fluent/plugin/output.rb).

## Tweaking elasticsearch plugins

After we understand how important buffer plugins is, we can go back to see how to tweak our elsticsearch plugin. For our use case, I try to collect logs as much as possible with small elasticsearch node.

The initial setting is like

```
 <buffer>
   @file
   path /fluentd/log/buffer

   # chunk + enqueue
   flush_mode interval
   flush_interval 1s

   flush_thread_count 2
   retry_type exponential_backoff
   retry_timeout 1h
   overflow_action drop_oldest_chunk
 </buffer>
```

The problem is that chunk fluentd collects is too small which lead to invoke too many elasticsearch write APIs. This also makes fluend queues many chunks in the disk due to fail requests of elasticsearch.

From AWS ES [doc](https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/aes-limits.html) we know that the http payload varies with different instance type. The maximum size of HTTP request payloads of most instance type is 100MB. Thus we should make our chunk limit size bigger but less than 100MB. Plus we should increase the flush_interval so that fluentd is able to create big enough chunk before flushing to queue. Here we also adjust flush_thread_count depending on elasticsearch plugin suggestion.

The modified version:

    <buffer>
      	@file
      	path /fluentd/log/buffer

      	total_limit_size 1024MB
      	# chunk + enqueue
        chunk_limit_size 16MB
        flush_mode interval
        flush_interval 5s
        # flush thread
        flush_thread_count 8

        retry_type exponential_backoff
        retry_timeout 1h
        retry_max_interval 30
        overflow_action drop_oldest_chunk
    </buffer>
### Result

After I change the setting, fluentd aggregator no longer complains about the insertion errors and drops the oldest chunks.
As you can see the following pictures show the memory usage drops dramatically so that it proves that fluentd works perfectly.

![](./before.png)

![](./after.png)

# References

1. [Fluentd Webinar: Best kept secret to unify logging on AWS, Docker, GCP, and more!](https://www.youtube.com/watch?v=aeGADcC-hUA)

```
Logging directly from microservice makes log storages overloaded
  - too many connections
  - too frequent import API call

Aggregation server
  - make log infra more reliable and scalable
  - connection aggregation
  - buffering for less frequent import API calls
  - data persistence during downtime
  - retry & recovery from down time
```

2. https://docs.fluentd.org/v1.0/articles/buffer-plugin-overview
3. https://github.com/uken/fluent-plugin-elasticsearch
4. https://gist.github.com/sonots/c54882f73e3e747f4b20
5. https://github.com/fluent/fluentd/blob/3566901ab4a00e0168b4a6078153dde85601fc53/lib/fluent/plugin/buffer.rb
6. https://abicky.net/2017/10/23/110103 Very detailed explanation how buffer works
