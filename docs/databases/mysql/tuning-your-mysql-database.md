---
author:
  name: Elle Krout
  email: ekrout@linode.com
description: 'Reviewing your MySQL configuration for optimum resource usage'
keywords: 'mysql,mysqltuner,resource tuning'
license: '[CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0)'
modified: Friday, February 27th, 2015
modified_by:
  name: Linode
published: 'Friday, February 27th, 2015'
title: Tuning Your MySQL Database
external_resources:
 - '[MySQL Documentation Library](http://dev.mysql.com/doc/index.html)'
 - '[MySQL Tuning Server Parameters](http://dev.mysql.com/doc/refman/5.7/en/server-parameters.html)'
 - '[MySQLTuner](http://mysqltuner.com/)'
---

Running MySQL at optimal settings for specific resources helps in handling larger server loads and prevents any server slow-down. Generally, after [tuning Apache](/docs/websites/apache-tips-and-tricks/tuning-your-apache-server) to handle larger loads it is beneficial to tune MySQL to additional connections.

Database tuning is an expansive topic, and this guide covers only the basics of editing your MySQL configuration.

Large MySQL databases can require a considerable amount of memory. For this reason, we recommend using a [high memory Linode](https://www.linode.com/pricing#high-memory) for such setups. 

{: .note}
>
>The steps in this guide require root privileges. Be sure to run the steps below as **root** or with the `sudo` prefix. For more information on privileges see our [Users and Groups](/docs/tools-reference/linux-users-and-groups) guide.

##Tools

In order to determine if your MySQL database needs to be reconfigured, it is best to look at how your resources are performing now. This can be done with the [top command](/docs/uptime/monitoring/top-htop-iotop) or with the Linode [Longview](/docs/platform/longview/longview) service. At the very least, you should familiarize yourself with the RAM and CPU usage of your server, which can be discovered with these commands:

	echo [PID]  [MEM]  [PATH] &&  ps aux | awk '{print $2, $4, $11}' | sort -k2rn | head -n 20
	ps -eo pcpu,pid,user,args | sort -k 1 -r | head -20

###MySQLTuner

The [MySQLTuner](http://mysqltuner.com/) script assesses your MySQL installation, and then outputs suggestions to increase your server's performance and stability.

1.  Download and run MySQLTuner:

		curl -L http://mysqltuner.pl/ | perl

2.  It outputs your results:

		 >>  MySQLTuner 1.4.0 - Major Hayden <major@mhtx.net>
		 >>  Bug reports, feature requests, and downloads at http://mysqltuner.com/
		 >>  Run with '--help' for additional options and output filtering
		Please enter your MySQL administrative login: root
		Please enter your MySQL administrative password:
		[OK] Currently running supported MySQL version 5.5.41-0+wheezy1
		[OK] Operating on 64-bit architecture

		-------- Storage Engine Statistics -------------------------------------------
		[--] Status: +ARCHIVE +BLACKHOLE +CSV -FEDERATED +InnoDB +MRG_MYISAM
		[--] Data in InnoDB tables: 1M (Tables: 11)
		[--] Data in PERFORMANCE_SCHEMA tables: 0B (Tables: 17)
		[!!] Total fragmented tables: 11

		-------- Security Recommendations  -------------------------------------------
		[OK] All database users have passwords assigned

		-------- Performance Metrics -------------------------------------------------
		[--] Up for: 47s (113 q [2.404 qps], 42 conn, TX: 19K, RX: 7K)
		[--] Reads / Writes: 100% / 0%
		[--] Total buffers: 192.0M global + 2.7M per thread (151 max threads)
		[OK] Maximum possible memory usage: 597.8M (60% of installed RAM)
		[OK] Slow queries: 0% (0/113)
		[OK] Highest usage of available connections: 0% (1/151)
		[OK] Key buffer size / total MyISAM indexes: 16.0M/99.0K
		[!!] Query cache efficiency: 0.0% (0 cached / 71 selects)
		[OK] Query cache prunes per day: 0
		[OK] Temporary tables created on disk: 25% (54 on disk / 213 total)
		[OK] Thread cache hit rate: 97% (1 created / 42 connections)
		[OK] Table cache hit rate: 24% (52 open / 215 opened)
		[OK] Open file limit used: 4% (48/1K)
		[OK] Table locks acquired immediately: 100% (62 immediate / 62 locks)
		[OK] InnoDB buffer pool / data size: 128.0M/1.2M
		[OK] InnoDB log waits: 0
		-------- Recommendations -----------------------------------------------------
		General recommendations:
		    Run OPTIMIZE TABLE to defragment tables for better performance
		    Enable the slow query log to troubleshoot bad queries
		Variables to adjust:
		    query_cache_limit (> 1M, or use smaller result sets)

	MySQLTuner offers suggestions regarding how to better the database. If you are wary about updating your database on your own, going by MySQLTuner's suggestions is one of the safer ways to improve your database performance.

##Tuning MySQL
When altering the MySQL configuration, be aware of the changes and how they affect your database. Even when following the instructions of programs such as [MySQLTuner](#mysqltuner), it is best to have some understanding of the process.

The file you are changing is located at `/etc/mysql/my.cnf`.

{: .note}
>
>Prior to updating the MySQL configuration, create a backup of the `my.cnf` file:
>
>	cp /etc/mysql/my.cnf ~/my.cnf.backup
>
>Best practices suggest that you make small changes at a time and monitor the server after each change. After each change restart MySQL:
>
>-	On Debian/Ubuntu:
>
>		service mysql restart
>
>-	On CentOS/Fedora:
>
>		/etc/init.d/mysqld restart
>
>When changing values in the `my.cnf` file be sure that the line you are changing is not commented out with the pound (`#`) prefix.

####key_buffer
Changing the `key_buffer` allocates more memory to MySQL, which can substantially speed up your databases, assuming you have the memory free. The `key_buffer` size should generally take up no more than 25% of the system memory when using the MyISAM table engine, and up to 70% for InnoDB. If the value is set too high, resources are wasted. According to MySQL's documentation, for servers with 256MB of RAM (or more) with many tables, a setting of 64M is recommended, while servers with 128MB of RAM and fewer tables can be set to 16M, the default value. Websites with even fewer resources and tables can have this value set lower.

####max_allowed_packet
The maximum size of a packet allowed to be sent. A packet is a single SQL state, a single row being sent to a client, or a log being sent from a master to a slave. If you know that your MySQL server is going to be processing large packets, it is best to raise this to the size of your largest packet. Should this value be set too small, you would receive an error in your error log.

####thread_stack
This value contains the stack size for each thread. MySQL considers the default value of the `thread_stack` variable sufficient for normal use; however, should an error relating to the `thread_stack` be logged, this can be raised.

####thread_cache_size
If `thread_cache_size` is "turned off" (set to 0), then all new connections being made need a new thread created for them, and when the connections disconnect the thread is destroyed. Otherwise, this value sets the number of unused threads to store in a cache until they need to be used for a connection. Generally this setting has little affect on performance, unless you are receiving hundreds of connections per minute, at which time this value should be rasied so the majority of connections are being made on cached threads.

####max_connections
Sets the maximum amount of *concurrent* connections. It is best to consider the maximum amount of connections you have had in the past and then setting this so there is some buffer between that number and the `max_connections` value. Note, the this does not mean the maximum amount of *users* on your website at one time, but the maximum amount of users making *requests* concurrently.

####table_cache
Should be kept higher than your `open_tables` value. To determine this value use:

	SHOW STATUS LIKE 'open%';
