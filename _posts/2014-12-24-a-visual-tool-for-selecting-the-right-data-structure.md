---
layout: post
title: A Visual Tool for Selecting the Right Data Structure
permalink: /blog/selecting-the-right-data-structure
comments: True
---
One of the first decisions every programmer has to make is *how to represent the data?* The answer, of course, depends on a lot of factors including the operations that will be performed on the data. A [data structure](http://en.wikipedia.org/wiki/Data_structure) is a particular way of organizing data in a computer so that it can be used efficiently. For example, if an applications requires lots of lookup by key operations a [Hash Table](http://en.wikipedia.org/wiki/Hash_table) could be a great option and if all you need is simple indexing you could use an [Array](http://en.wikipedia.org/wiki/Array_data_structure) instead.

<aside style="text-indent:-8px">* This is particularly important when dealing with large data since the time complexity of most operations is a function of the size of the problem.</aside>

Selecting the right data structure for an application can have huge performance implications.* Each data structure is good at supporting certain operations but not the others and understanding these trade-offs is essential for picking the right data structure.

I have created a simple tool (that I am calling [Big-O Selector](/bigoselector/), inspired by [Big-O Cheat Sheet](http://bigocheatsheet.com/)) to make selecting the right data structure easier.

[![Big-O Selector screenshot](/images/big-o-selector.png "Big-O Selector")](/bigoselector/)

I used a [parallel coordinates](http://en.wikipedia.org/wiki/Parallel_coordinates) plot to visualize all the supported operations by data structures. Each coordinate represents one operation and all of the operations supported by a given data structure are connected by a line. This succinct view makes the trade-off between data structures very apparent and allows us to select the best one for a given application. 

Head over to [Big-O Selector](/bigoselector/) and play with it yourself.