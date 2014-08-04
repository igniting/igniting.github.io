---
layout: post
title: "Progress so far"
description: ""
category: gsoc2014 
tags: []
---
{% include JB/setup %}

In my [last post]({% post_url 2014-07-08-solving-linear-equations %}), I had described that I needed to solve a set of linear equations to calculate the invidual cost factors. This approach has been described in a greater detail in comments of [MDEV-350](https://mariadb.atlassian.net/browse/MDEV-350). I will briefly describe the approach again.

We measure the time just before the query and after the query has completed, giving us the total time. Suppose we perform n type of operations in the query and we do the first operation a<sub>1</sub> times, the second a<sub>2</sub> times and so on. Let us denote the time taken by a single operation of first type by t<sub>1</sub>, of second type by t<sub>2</sub> etc. This gives us the equation,
> a<sub>1</sub>t<sub>1</sub> + a<sub>2</sub>t<sub>2</sub> + ... + a<sub>130</sub>t<sub>130</sub>= t<sub>total</sub>

where t<sub>total</sub> is the total time we calculated earlier. We have currently identified 4 kind of operations. `TIME_FOR_COMPARE`, `TIME_FOR_COMPARE_ROWID`, `READ_TIME` and `SCAN_TIME` are the constants associated with these operations. `READ_TIME` and `SCAN_TIME` are engine specific.

Note that most of the coefficients will be 0, as we have assumed a maximum of 64 engines (`MAX_HA`), and in a single equation, we will have coefficients from only a few engines. So, we needed to solve sparse linear equations. One more problem was that we will always have an overdetermined system of equations (we don't know after how many queries does the system of linear equations become solvable).

I decided to use the least squares method to solve this overdetermined system. Since we have a sparse set of equations, I decided to use [lsqr](http://web.stanford.edu/group/SOL/software/lsqr/). lsqr provides a C implementation for the least squares, which can be used under the BSD License. I tested `lsqr` with simple sparse linear equations, and it gave correct results. For storing coefficients of linear equations, I am using `std::map` instead of an array. Then I tested it with the data-set gathered by running `mtr`. It took less than 1s to solve a system of 30000 equations using lsqr. But the problem is I get negative values for the time values. One problem with the coefficients gathered is that we have different times for a same set of coefficients. I suspect there are few more bugs in the coefficients gathering part. You can see the coefficients [here](https://drive.google.com/file/d/0B7NiQb4EbbUVbUZsUU5EQTR6bFU/edit?usp=sharing) and the result obtained by lsqr [here](http://pastie.org/9444593).

Currently, we solve the equations before the THD disconnects or we have a maximum of 20 equations (value decided arbitarily). We store the total operations, total time and total time squared for each constant in a global object. Values from THD are added to the global object on THD disconnect, i.e. we increase, for each constant, total time and total operations. These values from the global object are written to the system table `optimizer_cost_factors` when mysqld exits. We populate the global object on server start with the values from the system table. We are currently calculating the new value by `total_time/total_ops`. Optimizer will now use these updated values instead of the default hard coded values.

The code is available on my [github repository](https://github.com/igniting/server/tree/selfTuningOptimizer).
