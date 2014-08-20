---
layout: post
title: "Documentation"
description: ""
category: gsoc2014 
tags: []
---
{% include JB/setup %}

In this post I will give a documentation for the work I have done. The description for aim of this project is given in detail in [MDEV-350](https://mariadb.atlassian.net/browse/MDEV-350) and in my earlier posts, so I am going to skip those here.

####Table of Contents
* <a href="#1">Getting the source code</a>
* <a href="#2">Interface for using cost factors</a>
* <a href="#3">Adding a new cost factor</a>
* <a href="#4">Under the hood</a>
* <a href="#5">Future Work</a>
* <a href="#6">Credits</a>

<h4 id="1">Getting the source code</h4>

This code is currently not merged in `mariadb`. You can get it from my [github repository](https://github.com/igniting/server/tree/selfTuningOptimizer).

<h4 id="2">Interface for using cost factors</h4>

I have added two new files `sql/opt_costmodel.h` and `sql/opt_costmodel.cc`. The class of interest is `Cost_factors`. This class is used to store all the cost factors (both global and engine specific). We have `Global_cost_factors` class for storing global cost factors and similarly `Engine_cost_factors` for engine specific factors.

In the current code, we have 2 global cost factors: `time_for_compare` and `time_for_compare_rowid` and two engine specific cost factors `read_time` and `scan_time`. These factors are accessible via a global `Cost_factors` object `cost_factors`. So you can get `time_for_compare` by calling `cost_factors.time_for_compare()`. For engine specific factors, the syntax is `cost_factors.read_time(handler *)`.

The values of these cost factors are initialized from the `optimizer_cost_factors` table of `mysql` database, when the server starts. If the cost factor is not present, a default value is given (for example `time_for_compare` has a default value of 5, and that of `time_for_compare_rowid` is 500).

<h4 id="3">Adding a new cost factor</h4>

Suppose you want to add a new engine specific cost factor, say `new_engine_factor`. Here is a step by step guide for doing that:

1. In `sql/opt_costmodel.h`, each constant has been given a number so that easy to store them in an array (or whatever). Give your constant a new number, like `#define NEW_ENGINE_FACTOR 2`. Also increase the value of `MAX_ENGINE_CONSTANTS` by 1 and `MAX_CONSTANTS` by 64 (`MAX_HA`). `MAX_CONSTANTS` is used while solving equations.
2. In the class `Engine_cost_factors` add a new variable for your constant, for example `Cost_factor new _engine_factor`. 
3. Give your cost factor a default value, for example `static const double DEFAULT_NEW_ENGINE_FACTOR= 1`.
4. Also, in the `optimizer_cost_factors` table, each constant is stored by it's name, so we need a mapping of variable to it's name. In the method `set_all_names()` of `Engine_cost_factors`, add a new entry for your variable. Example `all_names[YOUR_VARIABLE_NUMBER] = st_factor("NEW_ENGINE_FACTOR", &new_engine_factor)`.
5. You will also need to modify the method `update_engine_factor`. It has two variants, you need to modify both of them.
6. The last step is to expose your cost factor. In the `Cost_factors` class, add a new method `double new_engine_factor(handler *)` which will return the value of this factor based on the engine index.

Adding a new global cost factor is almost similar. (One important point to note is to increase the value of `MAX_CONSTANTS` by 1 and not by 64).

<h4 id="4">Under the hood</h4>

So you want to understand how are these factors being calculated.
As I had mentioned in the first section that the cost factors values are initialized from the `optimizer_cost_factors` table. This table currently has 5 columns, `const_name`, `engine_name`, `count`, `sum`, `sum_squared`. (The schema is in `scripts/mysql_system_tables.sql`). Engine name should be an empty string for global cost factors. The value for a cost factor is calculated as `sum/count`. Let's understand what are these `sum` and `count` in more detail.

Apart from a global `Cost_factors` object, `cost_factors`, each THD also has it's own copy of `Cost_factors`, `thd_cost_factors`. In each `THD` we measure the `query_time` for each query and how many times an operation related to a cost factor was performed. So for each query we have a linear equation relating `cost_factors` value, number of operations related to `cost_factor` and total time. This has been discussed in a greater detail in my earlier [blog post]({% post_url 2014-08-04-progress-so-far %}). We solve for cost factors when the `THD` disconnects or when we have `MAX_EQUATIONS`. I am using `lsqr` for solving sparse linear equations. These values are written to the global `cost_factors` object. The value is added to `sum` and the `count` is increased by 1 (Hence the equation `value=sum/count`). The values in the global `cost_factors` object are written to the `optimizer_cost_factors` table when the server disconnects. You might want to check out `THD::solve_equation` and `THD::build_equation` in `sql/sql_class.cc`.

<h4 id="5">Future work</h4>

* Since we have added only 4 cost factors, there is a lot of unaccounted time in each query. As a result of this, we get negative values for some of the coefficients when solved using `lsqr`.
* The framework has not been tested completely. So, if you get an error while experimenting with it, please report to me. I will try to add more test cases.

<h4 id="6">Credits</h4>

The entire project was an idea of Sergei Golubchik and he was also my mentor for this project. I will like to thank him for his constant guidance and encouragement.
