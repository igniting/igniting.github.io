---
layout: post
title: "More Coding"
description: ""
category: gsoc2014 
tags: []
---
{% include JB/setup %}

*You can find the code at [github](https://github.com/igniting/server/commits/selfTuningOptimizer)*.

In the [previous blog post]({% post_url 2014-05-25-coding-things-up %}), I had described how I had created a table in the `mysql` database to store cost factors persistently and written an interface to read the values from the table.

This week I have rewritten the whole thing as per my mentor's suggestions. The first important point was that I should read from the table only once, when the server starts. For this I had a look into the file `sql/mysqld.cc`. `mysqld_main()` is the entry point, which starts the server. There are functions like `udf_init()`, `tz_init()`, `acl_init()` etc. which do the same thing which I was supposed to implement. Following these examples, I have written a function `Cost_factors::init()`, which reads the cost factors from the table `optimizer_cost_factors` and populates the values of the data members of the class `Cost_factors`.

I have modified the `handler::read_time()` and `handler::scan_time()` functions to include the `READ_TIME_FACTOR` and `SCAN_TIME_FACTOR` respectively. I have tried writing a test which tries to test these factors. However I was not able to come up with an example query which will demonstrate that changing these factors result in a different execution plan.

The next part of the project would be to measure the change in the query times by modifying the cost factors and write the stats in the table. I will have a discussion about this with my mentor and give an update of the progress in the next week's blog.

