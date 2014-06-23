---
layout: post
title: "Work before mid term"
description: ""
category: gsoc2014 
tags: []
---
{% include JB/setup %}

So we have mid-term evaluations this week and I'll like to give a summary of whatever work has been done till now and what more is coming.

* There are two types of constants currently: Engine Specific, which will be different for different engines and Global, which are same across all engines. Currently in the code, I have two Engine specific constants: `read_time` and `scan_time`. We also have two Global constants `time_for_compare` and `time_for_compare_rowid`.

* Currently there is one global `Cost_factors` object, in which the data is populated on server start from the `optimizer_cost_factors` table present in the `mysql` database. The case when an engine is loaded after server start has been accounted for.

* There is one `Cost_factors` object present in each `THD`. The stats from queries are collected in a `measurement` object. This approach is discussed in more detail in [MDEV-350](https://mariadb.atlassian.net/browse/MDEV-350). Two approaches have been discussed there, I have started with the first approach. In the first approach we will be solving a system of linear equations to get individual cost factors.

* We will write the per THD stats to global stats, when the thread disconnects. This has not yet been implemented yet.

* The code is available at [my github repo](https://github.com/igniting/server/tree/selfTuningOptimizer). The implementation of `read_time` and `scan_time` factors have been tested.

Reviews/suggestions on the progress are welcome.
