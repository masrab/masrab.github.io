---
layout: post
title: A Visual Tool for Selecting the Right Data Structure
permalink: /blog/selecting-the-right-data-structure
comments: True
---

One of the first decisions every programmer has to make is *how to represent the data?* The answer, of course, depends on a lot of factors including the operations that will be performed on the data. A [data structure](http://en.wikipedia.org/wiki/Data_structure) is a particular way of organizing data in a computer so that it can be used efficiently. For example, if an applications requires lots of lookup by key operations a [Hash Table](http://en.wikipedia.org/wiki/Hash_table) could be a great option and if all you need is simple indexing you could use an [Array](http://en.wikipedia.org/wiki/Array_data_structure) instead.

<aside style="text-indent:-8px">* This is particularly important when dealing with large data since the time complexity of most operations is a function of the size of the problem`n`.</aside>

Selecting the right data structure for an application can have huge performance implications.* Each data structure is good at supporting certain operations but not the others and understanding these trade-offs is essential for picking the right data structure.

I have created a simple tool (that I am calling [Big-O Selector](/bigoselector/), inspired by [Big-O Cheat Sheet](http://bigocheatsheet.com/)) to make selecting the right data structure easier.

[![Big-O Selector screenshot](/images/big-o-selector.png "Big-O Selector")](/bigoselector/)

I am using a [Parallel coordinates](http://en.wikipedia.org/wiki/Parallel_coordinates) plot to visualize the supported operations by data structures. Each coordinate represents one of the operations and each line connects the operations supported by a given data structure.

Visualizing data structures in this way makes it very easy to see how each data structure ranks among others with respect to each operation. Head over to [Big-O Selector](/bigoselector/) and play with it yourself. 


## Getting the Data

I used [D3] to create the visualization. view the source [here].
D3 needs data
I got the data from ...

{% gist ec5f4627564b87afb349 %}

<aside>For a simple case like this, you could even copy and paste the data into your favorite spreadsheet software.</aside>

The tables are nested and the result from this simple scraping script won't be perfect but it gets us very close to what we need.


* Data collection/preparation: D3 stands for Data Driven Documents and what it means is that all the visual elements that are created are linked to some underlying data that defines them.

D3 provides helper functions to convert tabular data (e.g. in CVS format) to properly formatted JSON data that is needed by most D3 functions.


* read csv data and convert into JSON format. data will be an array of JavaScript objects like this:
{json sample}

* Create an axis for each dimension in the data which in our case means for each of the supported operations.

* Create a line for each data structure (i.e. one line per object in our original data array). We use [D3's data binding mechanism] and its [SVG path data generator] to create the line elements.

* 


## D3 Implementation

This is not D3 tutorial but at a high level, I took the following step to create the visualization:

