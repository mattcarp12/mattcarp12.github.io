---
layout: post
title: "Karatsuba Multiplication Part 1 - Algorithm Analysis"
date: 2022-02-13
categories: algorithms
tags: algorithms complexity analysis
usemathjax: true
---

[Karatsuba Multiplication](https://en.wikipedia.org/wiki/Karatsuba_algorithm) is an algorithm for multiplying two integers. Compared to the "grade-school" method, Karatsuba is much quicker when operating on large numbers.

## The Algorithm
Karatsuba multiplication is a recursive divide-and-conquer algorithm. Your two operands are split in half until the base case is reached, which is when they reach a single digit. Note that this algorithm will work for any base (e.g. binary base-2, octal base-8, etc.), but for simplicity I will use decimal base-10 for the remainder of this post.

Here is the algorithm:

1. Given two $$n$$ digit integers, $$x$$ and $$y$$, you can always represent them as such
    
    $$x = x_1B^m + x_0, \\ y = y_1B^m + y_0$$

    Where $$m$$ is a positive integer less than $$n$$. The algorithm is most efficient if you choose $$m = \lceil n / 2 \rceil$$.

2. Recursively calculate the following values:
   
   $$ z_0 = x_0y_0 \\ z_2 = x_1y_1 \\ z_1 = (x_1 + x_0)(y_1 + y_0) - z_2 - z_1 $$

3. Your final answer is 
   
   $$ xy = z_2B^{2m} + z_1B^m + z_0$$

Note that the above algorithm only requires 3 multiplications, and a few addition and subtraction operations. This is the key to Karatsuba's speed.

Now that we've seen the algorithm, let's analyze it!

## Algorithm Analysis
When doing algorithmic analysis, we are generally concerned with "worst-case analysis", which is to say we want to give bounds on the algorithm's runtime for **any** input. In order to facilitate worst-case analysis, we will make certain "relaxing assumptions".

The first relaxing assumption we make is that the length of our two operands $$x$$ and $$y$$ is some power of 2. That is, $$n = 2^k$$ for some $$k$$. This is fine because we can always "zero-pad" our numbers to make them a power of 2.

Now, for each level of recursion, we need to do a certain amount of work, namely additions and shifting (aka zero-padding). Each addition takes time proportional to the number of digits in the integer, $$n$$, and shifting takes constant time with respect to $$n$$. Thus, the extra work can be expressed as $$cn + d$$ for some constants $$c$$ and $$d$$.

Now we're ready to write our recurrence relation. Keeping in mind the fact that every recursion invokes 3 new multiplications, plus the extra work $$cn + d$$, our recurrence relation is

$$ T(n) = 3T(n/2) + cn + d$$

Since we know the $$cn$$ factor will dominate the constant factor, we disregard $$d$$ at this point (this is yet another relaxing assumption).

From here we could use the [Master Theorem](https://en.wikipedia.org/wiki/Master_theorem_(analysis_of_algorithms)) to solve this recurrence relation, but what fun is that? Let's use the [recurrence tree method](https://www.geeksforgeeks.org/how-to-solve-time-complexity-recurrence-relations-using-recursion-tree-method/) and flex our math skills in the process.

First, how tall is our tree? Well we assumed $$n = 2^k$$, and we know the recursion stops when $$n = 1$$, so the height of our tree is 

$$\frac{n}{2^i} = 1 \\ \Rightarrow i = \log_2 n$$

Now we need to sum up all the work done at every level of the recursion tree:

$$
\sum_{i = 0}^{\log_2{n} - 1} 3^ic\Big(\frac{n}{2^i} \Big) = cn\sum_{i=0}^{\log_2{n} - 1} \Big(\frac{3}{2}\Big)^i \\
= cn\Bigg[\frac{\Big(\frac{3}{2} \Big)^{\log_2 n}-1}{\frac{3}{2} - 1} \Bigg] \enspace (*) \\
= 2cn\Big[n^{\log_2(3/2)} - 1 \Big] \enspace (**) \\
= O(n^{\log_2 3}) \\ \approx O(n^{1.585})
$$

Where $$(*)$$ follows from geometric sum formula, and $$(**)$$ follows from a logarithm rule that says $$x^{\log_b y} = y^{\log_b x}$$.

This is much preferable to $$O(n^2)$$ run time of the "grade school" multiplication algorithm.