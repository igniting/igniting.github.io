---
layout: post
title: "First steps"
description: ""
category: gsoc2014
tags: []
---
{% include JB/setup %}

The first step was to get the source code using bazaar (version control used by mariadb). I spent my initial days in figuring out which part of code does what, and what part would I be working upon. The following resources were of great help:
* Although old and for MySQL, the [guided tour of the MySQL source code](https://dev.mysql.com/doc/internals/en/guided-tour.html) did a great job in getting me started with the source code.
* The book [Understanding MySQL Internals](http://shop.oreilly.com/product/9780596009571.do) also covers the internals of MySQL in great detail. This book also has a section on Optimizer.
* I wanted to know more about how the Optimizer works, so I searched online to see if any books existed on this topic. Lucky for me, I found [Inside the SQL Server Query Optimizer](http://www.red-gate.com/community/books/inside-sql-server-query-optimizer). This book gives a great understanding of what an Query Optimizer does behind the scenes.
* MySQL team has been working on a [cost model project](http://mysqlserverteam.com/the-mysql-optimizer-cost-model-project/). It might prove useful for me to take hints from their work, now and then.

I have just scanned through the above resources and would love to hear about **more resources that would be useful for me**.

Coming back to the task in hand, I needed to come up with a pair of constants. I started looking at the constants listed in `sql/sql_const.h`. The MySQL code has more constants listed there (with possibly more description). I initially started to explore `HEAP_TEMPTABLE_CREATE_COST` and `DISK_TEMPTABLE_CREATE_COST`. However after discussion with serg (my mentor for this project), it would be better to start with constants, which upon changing would show up visible results in the query plan selection. So I will be working with two new constants (they are implicitly assumed to be 1.0 currently): `SCAN_TIME_FACTOR` and `READ_TIME_FACTOR`. The optimizer calls `handler::scan_time()` and `handler::read_time()` at several places. We will now be using `SCAN_TIME_FACTOR*handler::scan_time()` and `READ_TIME_FACTOR*handler::read_time()` instead.

I started to look up more `scan_time()` and `read_time()`, more on that in a little while. I was also exploring on how to store the constants persistently on a database, so that it can be accessed and modified by the server. I initially thought an InformationSchema table would work. However, after discussion with serg and more research, I came to know that I_S tables are used only for presentation purpose, from which user can get to know value of constants. Persistent tables are stored in the `mysql` database. I have started looking into `sql/statistics.cc` as an example for reading/writing data into a table (This file does operations on `table_stats`, `column_stats` and `index_stats` tables). I still need to look into the **exact structure for our constant table** (Still to be explored).

Coming back to our pair of constants, `scan_time()` and `read_time()` are virtual functions, part of handler class in `sql/handler.h`. Each of the storage engine can provide their own implementations of these functions. It is clear that we will need to have different values for `READ_TIME_FACTOR` and `SCAN_TIME_FACTOR` for different engines. **Would it be true for every constant (i.e. would we need to have different values for same constant over different storage engines)?**

Looking at the current implementation in MySQL, a new cost model API has been introduced. Relevant code is present in `sql/opt_costmodel.h` and `sql/opt_costmodel.cc`. More details are in [worklog 7182](http://dev.mysql.com/worklog/task/?id=7182). **I think we will also need a similar structure**. There should be an API in place to read the constants.  

I think I will conclude here. This blog has been more words and less code and I promise things would change drastically in coming weeks. I would welcome your comments, specially on the **highlighted** statements.
