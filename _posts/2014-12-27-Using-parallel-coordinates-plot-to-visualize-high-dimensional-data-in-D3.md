---
layout: post
title: Using Parallel Coordinates Plot to Visualize High Dimensional Data in D3
permalink: /blog/prallel-coordinates-plot-d3
comments: True
---
In a previous [post]({% post_url 2014-12-24-a-visual-tool-for-selecting-the-right-data-structure %}) I introduced [Big-O Selector](/bigoselector/)–a visual tool that helps you pick the right data structutre for your application. In this post I review the key steps in creating this visualization in D3.

## Getting the Data

D3 stands for [Data Driven Documents](http://d3js.org/) and what it means is that all the visual elements in a plot are linked to some underlying data that defines them. In other words, to create a visualization we first need properly formatted data and then we have to [join](http://bost.ocks.org/mike/join/) the data to the visual elements in the DOM.

For this visualization, I needed the time and space complexity of all the operations supported by different data structures. Luckily, the nice folks at [Big-O Cheat Sheet](http://bigocheatsheet.com/) have already captured everything we need in nicely formatted tables. To get the data I wrote a simple scraping script in Python (using the awesome duo [BeautifulSoup](http://www.crummy.com/software/BeautifulSoup/) and [Requests](http://docs.python-requests.org/en/latest/)) to get the data in CSV format. 

{% gist ec5f4627564b87afb349 %}

<aside>For a simple case like this, we could just copy and paste the data into our favorite spreadsheet software.</aside>

The original HTML tables have nested headings and the result from our simple scraping script won't be perfect but it gets us very close to what we need. D3 provides [helper functions](https://github.com/mbostock/d3/wiki/CSV) to read and convert this tabular data to properly formatted JSON data that is required by most D3 functions. The final data that we will be working with is an array of JSON objects, in a format like this:

{% highlight JSON %}
[  
   {  
      "Name":"Singly-Linked List",
      "Indexing (Average)":"O(n)",
      "Search (Average)":"O(n)",
      "Insertion (Average)":"O(1)",
      "Deletion (Worst)":"O(1)",
      "Indexing (Worst)":"O(n)",
      "Search (Worst)":"O(n)",
      "Insertion (Worst)":"O(1)",
      "Space":"O(n)"
   },
   {  
      "Name":"Hash Table",
      "Indexing (Average)":"Undefined",
      "Search (Average)":"O(1)",
      "Insertion (Average)":"O(1)",
      "Deletion (Worst)":"O(n)",
      "Indexing (Worst)":"Undefined",
      "Search (Worst)":"O(n)",
      "Insertion (Worst)":"O(n)",
      "Space":"O(n)"
   },
   {  
      "Name":"Binary Search Tree",
      "Indexing (Average)":"O(log(n))",
      "Search (Average)":"O(log(n))",
      "Insertion (Average)":"O(log(n))",
      "Deletion (Worst)":"O(n)",
      "Indexing (Worst)":"O(n)",
      "Search (Worst)":"O(n)",
      "Insertion (Worst)":"O(n)",
      "Space":"O(n)"
   }
]

{% endhighlight %}

## D3 Implementation

In a [parallel coordinates](http://en.wikipedia.org/wiki/Parallel_coordinates) plot, each dimension of the data is represented by an axis and each observation is visualized via a line passing through its corresponding values across different axes. In our case, we need to create one axis for each of the operations supported by data structures.

{% highlight js %}

// extract the name of the dimensions
var dimensions = d3.keys(dataset[0]).filter(function(d) {
    return d != "Name"});

// create an ordinal scale for each dimension
var y = {};

dimensions.forEach(function(d){ 
y[d] = d3.scale.ordinal()
    .domain(d3.set(dataset.map(function(row) { return row[d];})).values().sort()) // extract column
    .rangePoints([0,height-20])
});

// add a group element for each dimension.
var g = svg.selectAll(".dimension")
         .data(dimensions)
       .enter().append("g")
         .attr("class", "dimension")
         .attr("transform", function(d) { return "translate(" + x(d) + ")"; });

g.append("g")
   .attr("class", "axis")
   .each(function(d) { d3.select(this).call(axis.scale(y[d])); });

// add axis labels   
var labels = g.append("text")
   .style("text-anchor", "middle")
   .attr("y", -15)
   .attr('class', 'label')
   .text(function(d) { return d; });

{% endhighlight %}

Next, we need to create a line for each data structure (i.e. one line per object in our original data array). We use D3's data binding mechanism and its [SVG path data generator](https://github.com/mbostock/d3/wiki/SVG-Shapes#path-data-generators) to create the line elements.

{% highlight js %}

var line = d3.svg.line();


// return the path for a given data point
function path(d, dimensions) {
  return line(dimensions.map(function(p) { return [x(p), y[p](d[p])]; }));
}

// append path
var foreground = svg.append('g')
      .attr('class', 'foreground')
      .selectAll('path')
      .data(dataset)
      .enter()
      .append('path')
      .attr('d', function(d) { return path(d,dimensions); } );

// highlight the first path
d3.select('.foreground path').call(highlight);

// hover effect
foreground
.on('mouseover', function(d) { 
  d3.select(this).call(highlight);
});

{% endhighlight %}

I have only highlighted parts of the code here. You can find the complete implementation [on Github]({{site.github}}/bigoselector).
