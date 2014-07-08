---
layout: post
title: "Solving linear equations"
description: ""
category: gsoc2014 
tags: []
---
{% include JB/setup %}
As I had discussed in my last blog post, we now have two engine specific constants: `read_time` and `scan_time`, and two global constants `time_for_compare` and `time_for_compare_rowid`.
To determine the time taken by the operations related to each constants, we are counting how many times these operations are being performed. This has been implemented for the constants mentioned above.
After each query, we write the data gathered into a file. To avoid collisions, we maintain different files for different threads.
Next step is to solve for t<sub>1</sub>, t<sub>2</sub> ..., t<sub>130</sub> from the equation:
> a<sub>1</sub>t<sub>1</sub> + a<sub>2</sub>t<sub>2</sub> + ... + a<sub>130</sub>t<sub>130</sub>= t<sub>total</sub>

Note we have 130 constants, `(2 + 2*64)`, where 64 is maximum allowed engines. Next, I needed a set of equations. I gathered some data by running `mtr`. Most of the coefficients would be 0, as we would be only using a few engines.
I have started with the least squares method to solve these equations. Here is a simple octave script that does that for me,
{% highlight octave %}
A = dlmread('coefficients.txt', ' ');
X = A(:, end-1);
Y = A(:, end);
S = ols(Y, X);
dlmwrite('solutions.txt', S);
{% endhighlight %}

The results are not promising. I get negative values for some of the coefficients. This may be because of the outliers in my sample data, introduced because of a bug in my code. I will fix it soon and then try again. I will also try out other methods, some of which will be specific to the case of sparse matrix input. You can download the dataset from [here](https://drive.google.com/file/d/0B7NiQb4EbbUVNVJFZ2xkRVR3Ylk/edit?usp=sharing).

As usual, you can find the code at my [repo](https://github.com/igniting/server/tree/selfTuningOptimizer). Comments and suggestions are welcome.

