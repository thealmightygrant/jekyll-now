---
layout: post
title: WTF is Happening in my Kafka?
permalink: wtf-is-happening-in-my-kafka
tags: [kafka, deep dive]
---

I work a lot with Kafka. It's come time for me to actually understand what's going on under the hood. I've mostly avoided this (though we have had multiple major Kafka outages at my current job).

Regardless, I'll put down some thoughts here and update them as I learn more about the mysterious Kafka.

# Kafka Internals

Kafka stores messages as a distributed log. Each log is known as a topic. Each topic is broken down as a partition. Partitions are stored on the same broker (a.k.a. kafknode). Consumers can read from these topics and producers can write to these topics. The parallelism inherent to kafka is provided by its ability to be partitioned. This allows for multiple processes to be reading from (possibly) multiple brokers. NOTE: Remember that each broker maintains one or more partitions for each topic. Those are the basics.

Kafka also maintains the counter of the offset at which each consumer of each partition is at. You can think of this as the page number in the book that the person reading a book is at. But, it's kind of like someone got gifted an entire book series and because they can't all read the same book at the same time, each person is reading a different book, and we are (creepily) maintaining a little spreadsheet of where each person is at in each book...Sally (consumer 1 in consumer group `game-of-thrones`) is at page 100 in the book Game of Thrones, George (consumer 2 in consumer group `game-of-thrones`) is at page 430 in the book A Storm of Swords, and Larry (consumer 3 in consumer group `game-of-thrones`)is at page 35 in the book A Feast For Crows. In the hidden kafka topic `__consumer_offsets`, each of these offsets (or page numbers) is recorded and maintained, just in case, Sally, Larry, or George forgets the page at which they were reading their books.

There is no requirement that the number of consumers matches the number of brokers, but the maximum number of consumers of a topic is the number of partitions on that topic. So, brokers and partitions are independent, but number of consumers and number of partitions on a topic are dependent. In addition, at some point, it becomes disadvantageous to have too many partitions on the same broker, so it is good to have some multiple of partitions of the number of brokers in your kafka cluster.

Messages that are being produced should have a key associated with them. This key defines which partition the message ends up on. Messages with the same key will always end up on the same partition. If keys are not diverse enough, messages will be clustered on specific partitions, which will correlate to messages being stored on the same brokers. Kafka uses a hashing algorithm on top of the key to determine which partition each message up ends in.

Each Kafka topic maintains data for either some specific amount of time or until some amount of bytes have been reached in the topic. These are described as the retention period (in milliseconds) or the number of retention bytes. If the period or max retention size is hit, then messages will be deleted, with the oldest messages being deleted first.

Partitions are split into segments. The active segment is never deleted. Within a segment there are two files, the log and the index. [Inside of the segment log, each messages's value, offset, timestamp, key, message size, compression codec, checksum, and version are stored](https://thehoard.blog/how-kafkas-storage-internals-work-3a29b02e026]. Then, the segment index maps the overall offset to the position of the message in the segment log).

Because you want your data to survive the crash of a kafka broker (trust me it happens...), you want to set a replication factor for each topic. This is the number of times tthat each partition on each topic is replicated within the Kafka cluster. Kafka waits for a minimum number of these replicas to be available before it allows for consumers to read from a particular offset of a particular topic/partition. This is known as the minimum ISRs or minimum in sync replicas. This number must be less than or equal to the total replication factor, and it defines both the resiliency and the speed at which consumers operate.

Whenever a partition receives a request for a read or a write from a consumer or producer, it deals with the leader of that partition. The followers passively replicate data from the leader. If the leader fails, one of the followers will become the leader. Each broker serves as a leader for some partitions and a follower for others. Only, in-sync-replicas are eligible as leaders. However, this is configurable.


[Kafka Best Practices](https://blog.newrelic.com/engineering/kafka-best-practices/),

Take it easy,<br>
From your buddy,<br>
Grant
