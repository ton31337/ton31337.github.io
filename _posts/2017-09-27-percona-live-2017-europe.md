---
layout: post
title: Recap Percona Live 2017 Europe
categories:
- blog
---

One month ago I attended SRECon17 Europe in Dublin. You can read my humble [notes](http://donatas.net/blog/2017/09/02/srecon17-europe/) about it if you want, but it's not about that today.

I have never been in Percona Live conference, this time was first. And speaking frankly, this conference was way better than SRECon. I think it happened because I always expect more and more from SRECon and this was different.

Another one reason why I liked this conference is that we presented [Hostinger](https://www.hostinger.com/) and [000webhost](https://www.000webhost.com/) to the world. You can find our slides [here](https://speakerdeck.com/balys/scaling-a-million-databases-on-000webhost).

To recap what I found useful from this conference, read the list below:

* [PMM](https://www.percona.com/doc/percona-monitoring-and-management/architecture.html) (Percona Monitoring and Management) tool itself is more suited for small-medium workloads because as I mentioned in my previous blog [post](http://donatas.net/blog/2017/09/02/srecon17-europe/) _PS_ and _slow query_ adds some performance penalty. This tool actually works only with PS/SQ. Other tools like [VividCortex](https://www.vividcortex.com/) or LinkedIn (not yet open-sourced) does timing more granular and almost without losing performance. The main point achieving painless processing for queries is the agent living together with MySQL instance and sniffing the traffic using `libpcap`. The biggest drawback if using agent-based processing is the lack of detailed statistics: InnoDB status, rows examined, rows sent, etc. You could see only stats in terms of timing.
* InnoDB [Undo tablespace](https://dev.mysql.com/doc/refman/5.6/en/glossary.html#glos_undo_tablespace). This is mainly used for storing the previous state of the row for update/delete statements. Before 5.6 undo logs resided in the system tablespace. Now it's under separate undo tablespace.
* MySQL 5.7 data dictionary improvement for InnoDB. No more need to use `innodb_file_per_table=1` even if you have a plethora of databases/tables. Otherwise, you just give more work for the OS to handle those files.
* Another very interesting and promising feature for MySQL 8.0 are hints for queries. You can give hints for the query like `SELECT /* sort_buffer_size=1m */ 1;` to override `sort_buffer_size` or even other system variables for the query.
* As Peter Zaitsev said, Percona is doing already 10 years of cleanup.
* DAX and InnoDB support - sounds interesting, but no real-world cases still, only testing.
* There was an interesting topic about internal data structures from InnoDB, B+Tree, Adaptive Hash Index (it's just a decent order of the B+Tree for leaf nodes and it's done automatically), etc. I cannot find any resource to answer my question: How often is B+Tree rotated? Is there any delay involved or with every change? If you know the answer, please write [me](http://donatas.net/about/) back.
* InnoDB checkpointing with large log files could be painful, Percona recommends to set this up to 10GB max. Log files are actually like ring buffers. If you have small log file it means queries could be overwritten if not flushed to disk as quickly as possible.
* `innodb_buffer_pool_instances` distribute CPU affinity between pools but do not cross 16 threads, because performance degrades.
* "I could say Percona is like the operating system, but someone refers it to database".
* Disable change buffer if using SSDs.
* "Worse than useless".
* InnoDB is using row locking, but _all_ index entries are locked during a query. The bigger the index the worse the response time.
* The most trending open source products in this conference were: [ClickHouse](https://clickhouse.yandex/), [Vitess](http://vitess.io/), [ProxySQL](http://www.proxysql.com/), [Orchestrator](https://github.com/github/orchestrator), [MyRocks](http://myrocks.io/).
* Slack was presenting their usage of Vitess, how they migrated from application-based sharding to Vitess and so on. The biggest drawback is performance because of the intermediate layer (Vitess). It's mostly network-bound. The most exciting feature of Vitess is `hot row protection` to avoid sending N+1 identical requests to shards and thus sending the single one.
* Another very interesting and very technical was about MySQL replication. Turn off binlog on slaves to save replication lag. Since MySQL 5.7+ there are even more interesting approaches how to improve replication like: execute parallel replication if both transactions hold the same lock; delay group commits to poll for more transactions; write-based replication with `LOGICAL_CLOCK`; execute parallel replication if transactions are independent. I don't know if it's somehow possible to tell the query if it's dependent on others or not. Is it possible to fanout binlog to group replication nodes using multicast? What are the benchmark results if using busy polling for slaves?
* One more talk about load balancers, proxies. MySQL Router, MaxScale, HAProxy, Nginx, ProxySQL, Keepalived. I would add [ExaZK](http://donatas.net/blog/2017/02/26/exazk/) as a solution for master-master replication cases. It's more granular than keepalived because of locking, but it requires you more infrastructure/networking knowledge. By the way, MaxScale works only with 8 threads. With more, it just does not.
* State of the dolphin: customizable prompt, more `JSON_` functions (like `JSON_TABLE` to map JSON attributes as columns), common table expressions, hot row contention, invisible indexes, histograms)
* Wikipedia told what, how and why they are using a different kind of databases, tools to run their infrastructure at scale. Facebook, Google is not desired to this talk :D
