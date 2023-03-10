---
title: "[Summary] How Discord Stores Billions of Messages"
description: "Summary of why and how Discord migrates their database to Cassandra"
date: 2023-01-12T00:17:04+08:00
draft: false
tags: [database, mongodb, cassandra]
---

Link: [HOW DISCORD STORES BILLIONS OF MESSAGES](https://discord.com/blog/how-discord-stores-billions-of-messages)


The article discusses how Discord chose and migrated to new database in order to deal with tons of stored messages. The article was published in 2017 and provided insights about data storage. Having an idea about [how Cassandra read/write data](https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/dml/dmlIntro.html) is helpful to understand the content. Cassandra uses a storage structure similar to [Log-Structured Merge Tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree), which is known for its high performance in writing but less efficient in reading, comparing to [B-Tree](https://en.wikipedia.org/wiki/B-tree).

The following are main takeaways:
1. Know what you need. Discord took note of their read/write patterns, including traffics, read/write ratio of different kinds of channel. For example, voice chat heavy channel accumulates few messages, compared to text chat heavy channel.
2. List requirements and test candidates. A linearly scalable and fault-tolerant database help reduce the operation work. They have alerts go off when P95 of API’s response time goes above 80ms. For a messaging service, API's response time is critical.
3. They switched to Cassandra from MongoDB. Roughly speaking, Cassandra is a KKV store. The first K stands for the partition key, and the second K represents the clustering key, which identifies a row in a partition. Previously, Discord used a compound key (channel_id, created_at) in MongoDB. However, created_at isn't a good clustering key because there may be two messages that have same created time. Luckily, Discord had already applied Snowflake, which is chronologically sorted, for id generation. Therefore, the primary key became (channel_id, message_id).
4. Bucket the message by time. The size of partition is restricted due to GC pressure. Discord reset the primary key to ((channel_id, bucket), message_id). They looked at the biggest channel on Discord and decided a bucket/partition to hold messages of 10 days.
5. Optimize tombstone mechanism in Cassandra. In Cassandra, deleted data is not immediately removed from the disk. Instead, Cassandra writes a value, known as tombstone, to indicate the data is deleted. When performing reading, Cassandra reads and ignores the deleted data. The tombstones will be dropped and deleted data will be removed after compaction and the life span of tombstone expires. This causes issues when all of a sudden a lot of data is deleted in the partition. Cassandra needs to read a lot of tombstones to finally output results, resulting in great latency for reading. Therefore, Discord optimized the deleting/writing process to avoid tombstones overhead. They shortened the life span of tombstones, contributing to less tombstones that Cassandra meets during reading. To keep consistency of data, they ran Cassandra repairs in a regular basis. Additionally, Discord recorded empty partitions, which boosts read performance by ignoring useless partitions.
6. Discord performs the migration with only four backend engineers(no DevOps). I am impressed by how competent they are.
