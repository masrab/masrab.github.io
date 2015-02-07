---
layout: post
title: Analyzing Public Recipe Data in R
permalink: /blog/analyzing-public-recipe-data-in-R
comments: True
---

In this post, we are going to use a publicly available recipe dataset to answer one simple question: **Cuisine from which countries are the most similar?** There are of course various ways to answer this question but here I'll be using a rather simplistic view and focus only on the ingredients used in each recipe. Let's get started by loading and exploring our dataset.

## The Dataset
The dataset we are working with is a [collection recipes](http://yongyeol.com/data/scirep-cuisines-detail.zip) scraped from [epicurious.com](http://www.epicurious.com/) where each record contains the type of the cuisine (e.g. American, French, Chinese) and the ingredients that were used in the recipe.

The data is stored in flat text files where each line is one record. Here's the first two lines of the raw data file:

{% highlight text %}
[1] "Vietnamese\tvinegar\tcilantro\tmint\tolive_oil\tcayenne\tfish\tlime_juice\tshrimp\tlettuce\tcarrot\tgarlic\tbasil\tcucumber\trice\tseed\tshiitake"
[2] "Vietnamese\tonion\tcayenne\tfish\tblack_pepper\tseed\tgarlic"                                                                                     
{% endhighlight %}

Before we can use this data, we need to read the data into R and convert it to a data frame. First, we combine all the recipes from the same cuisine type together to create a food corpus.

{% highlight r %}
library(dplyr)
# read each line
dat <- readLines("../data/epic_recipes.txt") %>% strsplit(split = "\t")
# extract cuisine and ingredients
cuisine <- sapply(dat, function(x) x[1])
ingredients <- sapply(dat, function(x) paste(x[-1],collapse = " ")) 
# for each cuisine, join together the ingredients from all the recipes
cuisine_names <- cuisine %>% unique
names(cuisine_names) <- cuisine_names
food <- sapply(cuisine_names, function(x){
  ingredients[cuisine == x] %>% paste(collapse = " ")
  })
{% endhighlight %}

The next step is to create a feature matrix from this corpus. One simple way to create features is to represent each cuisine by the number of times each ingredient has been used in it. This is essentially the standard [_bag-of-words_](http://en.wikipedia.org/wiki/Bag-of-words_model) representation often used in text analytics.


{% highlight r %}
library(tm)
# convert text corpus to a document term matrix
food_dtm <- VectorSource(food) %>% VCorpus %>% DocumentTermMatrix
# create a data frame from the document term matrix
food_df <- data.frame(cuisine = cuisine_names, stringsAsFactors=FALSE, row.names=NULL) %>% 
  cbind(as.matrix(food_dtm)) %>% tbl_df
{% endhighlight %}
Here's what the data looks like:

{% highlight text %}
Source: local data frame [6 x 7]

                cuisine almond anise anise_seed apple apple_brandy apricot
1            Vietnamese      0     0          0     0            0       0
2                Indian      8     0          2     6            0       6
3    Spanish_Portuguese     27     1          3     3            0       1
4                Jewish     31     0          1    28            0      21
5                French     65    12          6    39           10      16
6 Central_SouthAmerican     13     0          5     0            0       0
{% endhighlight %}

The data is currently in what is known as *wide* format (i.e. each feature is represented as one column). To make our lives easier later on (when plotting the data), let's create a *tall* version of the data as well: 


{% highlight r %}
library(tidyr)
# "wide" --> "tall"
food_tall <- food_df %>% gather("ingredient", "count", -cuisine)
{% endhighlight %}

## Most Popular Ingredients
What are the most popular ingredients in each type of cuisine? We can use the feature matrix that we just created to answer this question. 


{% highlight r %}
library(ggplot2)

N_top <- 10
# calculate the ranking of each ingredient for each cuisine type
food_tall <- food_tall %>% group_by(cuisine) %>% 
  mutate(rank = min_rank(desc(count)))

food_tall %>%
  filter(rank<=N_top) %>% # top N ingredients
  ggplot +
  geom_point(aes(ingredient, cuisine, color = rank, size = rank)) +
  scale_color_gradient(name = "Popularity", breaks=1:N_top,
                       low="red", high="black") +
  scale_size(name = "Popularity", breaks=1:N_top, range=c(6,1)) + 
  guides(color = guide_legend()) + 
  theme(axis.text.x = element_text(angle = 90, hjust = 1, vjust = .5, size = 11))
{% endhighlight %}

![center](/../images/2015-01-03-nutrition/unnamed-chunk-7-1.png) 
Looking across columns, we see that onion, garlic, wheat and butter are among the most popular ingredients in almost all types of cuisine. Looking across rows, we see that Parmesan cheese, for example, is a popular ingredient for Italian cuisine only and not the others. 


We can also plot the number of unique ingredients found in each cuisine:


{% highlight r %}
recipe_count <- table(cuisine) %>% as.data.frame.table(responseName = "recipe_count")

# Number of unique ingredients
food_tall %>% 
  summarise(unique_ingredients = sum(count>0) ) %>% 
  arrange(desc(unique_ingredients)) %>% 
  mutate(cuisine = factor(cuisine,
                          levels = cuisine[order(rev(unique_ingredients))])) %>% 
  ggplot + geom_bar(aes(cuisine, unique_ingredients),
                    stat = "identity", fill = "gold",
                    color = "brown", width = .7) +
  ylab("Number of unique ingredients") +
  theme(axis.text.x = element_text(angle=90, vjust = .5, hjust = 1))
{% endhighlight %}

![center](/../images/2015-01-03-nutrition/unnamed-chunk-8-1.png) 
American, French and Asian foods have the highest number of unique ingredients. We may be tempted to use the number of unique ingredients as a measure of _complexity_ for each cuisine but that is not necessarily true. Let's take a look at the number of recipes we have for each cuisine and compare that against the number of unique ingredients: 


{% highlight r %}
food_tall %>% left_join(recipe_count) %>% 
  mutate(unique_ingredients = sum(count>0) ) %>% 
  arrange(desc(unique_ingredients)) %>% 
  qplot(recipe_count, unique_ingredients, data = .,
        size = I(3), color = I("brown")) +
  scale_x_log10() +
  xlab("Number of recipes in the dataset") +
  ylab("Number of unique ingredients")
{% endhighlight %}

![center](/../images/2015-01-03-nutrition/unnamed-chunk-9-1.png) 
There is a high correlation between recipe counts and the number of unique ingredients. So perhaps the reason some of the cuisines have fewer unique ingredients is because we just haven't seen collected enough recipes yet.

## Hierarchical Clustering
We now have everything we need to try and answer the question that motivated our analysis. We want to know which cuisine are the most similar in terms of their ingredients. We can use a clustering algorithm and divide our samples into different clusters in an unsupervised fashion. In a problem like this where we don't have a predefined number of clusters in mind, [hierarchical clustering](http://en.wikipedia.org/wiki/Hierarchical_clustering) is a good option. The algorithm starts by assigning each sample to its own cluster and then proceeds iteratively, at each stage joining the two most similar clusters, continuing until there is just a single cluster. 


{% highlight r %}
hc <- data.frame(food_df, row.names = "cuisine")

# without standardizing
hc %>% dist %>% hclust %>% plot(hang= -1, xlab = "", main="Clustering without standardization", sub="")
{% endhighlight %}

![center](/../images/2015-01-03-nutrition/unnamed-chunk-10-1.png) 

One way to visualize the clustering results is using a [dendogram](http://en.wikipedia.org/wiki/Dendrogram) (we can also use graphs as we will see next). The height of the dendogram is the distance (dissimilarity) between samples or joined clusters. By default, the `dist()` function in R uses a Euclidean distance measure which get dominated by the variables with highest variance. To deal with this, we need to standardize our data prior to calculating the dissimilarity matrix. Intuitively, each row of the standardized data will contain numbers between 0 and 1 that capture the contribution of each ingredient to a given cuisine.


{% highlight r %}
library(dendextend) # pretty dendograms

mar_orig <- par("mar")
par(mar=c(11,5,2,2)) # set the margines

# need to standardize rows prior to calculating dissimilarity
norm_rows <- function(mat){
  mat / rowSums(mat)
  }

# distance matrix
d <- hc %>% 
  norm_rows %>%
  dist

food_dend <- hclust(d) %>% as.dendrogram

num_clusters <- 4 # highlight four clusters
food_dend %>% set("branches_k_color", k = num_clusters) %>% plot(ylab="Distance")

# add bounding boxes
food_dend %>% rect.dendrogram(k=num_clusters, border = 8, lty = 5, lwd = 2, lower_rect=0)
{% endhighlight %}

![center](/../images/2015-01-03-nutrition/unnamed-chunk-11-1.png) 


Now, this is much better! It looks nicer and it makes much more sense. All the cuisine that we know as being similar are nicely grouped together in this dendogram. The hierarchy and how smaller clusters are joined to make bigger ones is also interesting. For example, we see that Moroccan and African foods are (obviously) very similar and together they form a cluster that is similar to Middle Eastern food. 

It is also interesting how French cuisine turns out to be the most similar to American cuisine. Sounds surprising? Remember that our analysis is based only on the ingredients used in the recipes and nothing else. We can examine the raw counts for the top ten ingredients to confirm:


{% highlight r %}
food_tall %>% filter(cuisine %in% c("American", "French")) %>%
  group_by(cuisine) %>% mutate(rank = min_rank(desc(count))) %>% 
  filter(rank <= 10) %>% 
  ungroup %>% spread(cuisine, count, fill="—") %>% 
  arrange(rank) %>% kable
{% endhighlight %}



|ingredient    | rank|American |French |
|:-------------|----:|:--------|:------|
|butter        |    1|2219     |488    |
|egg           |    2|1738     |433    |
|wheat         |    3|1574     |355    |
|olive_oil     |    4|1512     |307    |
|garlic        |    5|1267     |272    |
|cream         |    6|—        |269    |
|onion         |    6|1254     |—      |
|cream         |    7|1238     |—      |
|onion         |    7|—        |236    |
|black_pepper  |    8|1154     |205    |
|milk          |    9|942      |193    |
|parsley       |    9|—        |193    |
|vegetable_oil |   10|926      |—      |

The top ten ingredients in French and American cuisine are indeed very similar.

## Using D3 forced network to visualize the clusters

We can also use a graph to visualize the pairwise distance matrix that we calculated and used as input for clustering.

We are going to use a [force-directed network graph in D3](https://github.com/mbostock/d3/wiki/Force-Layout) for this visualization. Each node in the graph represents one cuisine and the length of the edges connecting these nodes is  proportional to the distance between nodes. To better "uncover" the hidden clusters in the data, I pruned the graph by removing the edges between nodes that are sufficiently far from each other (only keeping the top 40% of edges). 

The graph representation makes it easier to visually detect clusters in the data. For comparison, the nodes of the graph are colored based on the the four clusters that we previously identified using hierarchical clustering.




{% highlight r %}
library(d3Network)
nodes <- as.matrix(d) %>% 
  create_nodes(Group = cutree(food_dend, k = num_clusters))

links <-  as.matrix(d) %>% 
  create_links

# prune the graph. Remove 60% of the links based on distance.
links <- links %>% filter(Value < quantile(d,.4))

d3ForceNetwork(Links = links, Nodes = nodes, Source = "Source",
               Target = "Target", Value = "Value", NodeID = "Name",
               Group = "Group",
               linkDistance = "function(d) { return 1000 * d.value + 10; }",
               linkWidth = "function(d) { return 1/(10*d.value); }",
               width = 650, height = 500,
               charge = -250,
               opacity = 1, 
               standAlone = FALSE,
               parentElement = "div#cuisine-forcenet")
{% endhighlight %}

<div id="cuisine-forcenet"></div>

<style>
.link {
stroke: #666;
opacity: 1;
stroke-width: 1.5px;
}
.node circle {
stroke: #fff;
opacity: 1;
stroke-width: 1.5px;
}
.node:not(:hover) .nodetext {
display: none;
}
text {
font: 7px serif;
opacity: 1;
pointer-events: none;
}
</style>

<script src='http://d3js.org/d3.v3.min.js'></script>

<script> 
 var links = [ { "source" : 7, "target" : 0, "value" : 0.0757184368151617 }, { "source" : 24, "target" : 0, "value" : 0.094238434136986 }, { "source" : 5, "target" : 1, "value" : 0.119801962527452 }, { "source" : 7, "target" : 1, "value" : 0.101979873387097 }, { "source" : 11, "target" : 1, "value" : 0.0894080317707846 }, { "source" : 12, "target" : 1, "value" : 0.113576340582768 }, { "source" : 21, "target" : 1, "value" : 0.108800326629006 }, { "source" : 24, "target" : 1, "value" : 0.124287029470481 }, { "source" : 25, "target" : 1, "value" : 0.124477510324697 }, { "source" : 3, "target" : 2, "value" : 0.118834766034534 }, { "source" : 4, "target" : 2, "value" : 0.101874484189859 }, { "source" : 5, "target" : 2, "value" : 0.0869989089736921 }, { "source" : 6, "target" : 2, "value" : 0.0912146379693411 }, { "source" : 9, "target" : 2, "value" : 0.0864930881386202 }, { "source" : 10, "target" : 2, "value" : 0.0923212763297632 }, { "source" : 11, "target" : 2, "value" : 0.0845242855558932 }, { "source" : 12, "target" : 2, "value" : 0.0857655176168215 }, { "source" : 13, "target" : 2, "value" : 0.121440188289424 }, { "source" : 14, "target" : 2, "value" : 0.0801718283026827 }, { "source" : 16, "target" : 2, "value" : 0.10998438089799 }, { "source" : 19, "target" : 2, "value" : 0.0637522486691244 }, { "source" : 21, "target" : 2, "value" : 0.100321706184915 }, { "source" : 22, "target" : 2, "value" : 0.120104536911187 }, { "source" : 25, "target" : 2, "value" : 0.107618043416532 }, { "source" : 4, "target" : 3, "value" : 0.075792916452623 }, { "source" : 6, "target" : 3, "value" : 0.120603040949305 }, { "source" : 8, "target" : 3, "value" : 0.0933841146673712 }, { "source" : 9, "target" : 3, "value" : 0.124060560804078 }, { "source" : 10, "target" : 3, "value" : 0.0717932748430411 }, { "source" : 11, "target" : 3, "value" : 0.10910887176174 }, { "source" : 12, "target" : 3, "value" : 0.105992463450293 }, { "source" : 13, "target" : 3, "value" : 0.0723738673628801 }, { "source" : 14, "target" : 3, "value" : 0.117344852844308 }, { "source" : 15, "target" : 3, "value" : 0.0972714587148483 }, { "source" : 18, "target" : 3, "value" : 0.100016120363849 }, { "source" : 19, "target" : 3, "value" : 0.123034252551801 }, { "source" : 21, "target" : 3, "value" : 0.123362333134649 }, { "source" : 22, "target" : 3, "value" : 0.0945932293088356 }, { "source" : 23, "target" : 3, "value" : 0.098764890694245 }, { "source" : 5, "target" : 4, "value" : 0.123162095902027 }, { "source" : 6, "target" : 4, "value" : 0.109949359628071 }, { "source" : 8, "target" : 4, "value" : 0.0833934808002814 }, { "source" : 9, "target" : 4, "value" : 0.111775216476005 }, { "source" : 10, "target" : 4, "value" : 0.0417254348600908 }, { "source" : 11, "target" : 4, "value" : 0.114630965920653 }, { "source" : 12, "target" : 4, "value" : 0.106285992097309 }, { "source" : 13, "target" : 4, "value" : 0.0571997345139504 }, { "source" : 14, "target" : 4, "value" : 0.0883957020795666 }, { "source" : 15, "target" : 4, "value" : 0.0864043606152992 }, { "source" : 18, "target" : 4, "value" : 0.0850751222973209 }, { "source" : 19, "target" : 4, "value" : 0.108089445711319 }, { "source" : 22, "target" : 4, "value" : 0.0757163870683307 }, { "source" : 23, "target" : 4, "value" : 0.0783725046792344 }, { "source" : 6, "target" : 5, "value" : 0.0950078902423833 }, { "source" : 10, "target" : 5, "value" : 0.101967458249754 }, { "source" : 11, "target" : 5, "value" : 0.0937087235839511 }, { "source" : 12, "target" : 5, "value" : 0.105304013489878 }, { "source" : 14, "target" : 5, "value" : 0.118739058675046 }, { "source" : 16, "target" : 5, "value" : 0.0357220266526651 }, { "source" : 19, "target" : 5, "value" : 0.114744210355473 }, { "source" : 21, "target" : 5, "value" : 0.114931825859479 }, { "source" : 22, "target" : 5, "value" : 0.112942846373909 }, { "source" : 25, "target" : 5, "value" : 0.0499112786820937 }, { "source" : 10, "target" : 6, "value" : 0.0936336826378278 }, { "source" : 11, "target" : 6, "value" : 0.105978829595734 }, { "source" : 12, "target" : 6, "value" : 0.113670907419824 }, { "source" : 13, "target" : 6, "value" : 0.11794051075807 }, { "source" : 14, "target" : 6, "value" : 0.118781030327368 }, { "source" : 16, "target" : 6, "value" : 0.110817146339208 }, { "source" : 19, "target" : 6, "value" : 0.119884757049942 }, { "source" : 22, "target" : 6, "value" : 0.0972775480794915 }, { "source" : 25, "target" : 6, "value" : 0.109454753961223 }, { "source" : 24, "target" : 7, "value" : 0.0848167090429891 }, { "source" : 10, "target" : 8, "value" : 0.076297245548567 }, { "source" : 13, "target" : 8, "value" : 0.0720536144903007 }, { "source" : 15, "target" : 8, "value" : 0.091242450206029 }, { "source" : 18, "target" : 8, "value" : 0.0804704246722467 }, { "source" : 22, "target" : 8, "value" : 0.0940559955469568 }, { "source" : 23, "target" : 8, "value" : 0.0901931632162004 }, { "source" : 10, "target" : 9, "value" : 0.10685453723339 }, { "source" : 11, "target" : 9, "value" : 0.108249060023335 }, { "source" : 12, "target" : 9, "value" : 0.0789561077774746 }, { "source" : 14, "target" : 9, "value" : 0.0845121102840735 }, { "source" : 19, "target" : 9, "value" : 0.0521210723698305 }, { "source" : 21, "target" : 9, "value" : 0.111393219915476 }, { "source" : 11, "target" : 10, "value" : 0.0971951436681964 }, { "source" : 12, "target" : 10, "value" : 0.0924438619237694 }, { "source" : 13, "target" : 10, "value" : 0.0591824076824621 }, { "source" : 14, "target" : 10, "value" : 0.087716871761997 }, { "source" : 15, "target" : 10, "value" : 0.0865353131426039 }, { "source" : 18, "target" : 10, "value" : 0.0819139513574867 }, { "source" : 19, "target" : 10, "value" : 0.104197783693116 }, { "source" : 21, "target" : 10, "value" : 0.116734102985087 }, { "source" : 22, "target" : 10, "value" : 0.0584626283306495 }, { "source" : 23, "target" : 10, "value" : 0.0824905874686817 }, { "source" : 25, "target" : 10, "value" : 0.124838187274963 }, { "source" : 12, "target" : 11, "value" : 0.0587527553294329 }, { "source" : 14, "target" : 11, "value" : 0.110779505723334 }, { "source" : 16, "target" : 11, "value" : 0.110196821029092 }, { "source" : 19, "target" : 11, "value" : 0.0915640958996945 }, { "source" : 21, "target" : 11, "value" : 0.0440523943103762 }, { "source" : 22, "target" : 11, "value" : 0.123032625737172 }, { "source" : 25, "target" : 11, "value" : 0.104601850794568 }, { "source" : 13, "target" : 12, "value" : 0.121118823344596 }, { "source" : 14, "target" : 12, "value" : 0.0965958206755896 }, { "source" : 16, "target" : 12, "value" : 0.123957888063344 }, { "source" : 19, "target" : 12, "value" : 0.0698650578335563 }, { "source" : 21, "target" : 12, "value" : 0.0599884885040736 }, { "source" : 22, "target" : 12, "value" : 0.122006710954458 }, { "source" : 25, "target" : 12, "value" : 0.119544818374549 }, { "source" : 14, "target" : 13, "value" : 0.115705855394764 }, { "source" : 15, "target" : 13, "value" : 0.073795297267222 }, { "source" : 18, "target" : 13, "value" : 0.0675017624188165 }, { "source" : 22, "target" : 13, "value" : 0.0740078979842991 }, { "source" : 23, "target" : 13, "value" : 0.0688269888825127 }, { "source" : 19, "target" : 14, "value" : 0.0676105630316842 }, { "source" : 21, "target" : 14, "value" : 0.120625983910468 }, { "source" : 22, "target" : 14, "value" : 0.118415492341247 }, { "source" : 18, "target" : 15, "value" : 0.0899542094762093 }, { "source" : 22, "target" : 15, "value" : 0.0863234943407565 }, { "source" : 23, "target" : 15, "value" : 0.0699313404093615 }, { "source" : 25, "target" : 16, "value" : 0.0414656431388081 }, { "source" : 20, "target" : 17, "value" : 0.0924093707178076 }, { "source" : 24, "target" : 17, "value" : 0.0587153093195449 }, { "source" : 22, "target" : 18, "value" : 0.0873710924942921 }, { "source" : 23, "target" : 18, "value" : 0.0959537289863271 }, { "source" : 21, "target" : 19, "value" : 0.0939156814729362 }, { "source" : 24, "target" : 20, "value" : 0.0747173195715422 }, { "source" : 25, "target" : 21, "value" : 0.123658114127637 }, { "source" : 23, "target" : 22, "value" : 0.0835889717683904 } ] ; 
 var nodes = [ { "name" : "Vietnamese", "group" : 1 }, { "name" : "Indian", "group" : 2 }, { "name" : "Spanish_Portuguese", "group" : 3 }, { "name" : "Jewish", "group" : 4 }, { "name" : "French", "group" : 4 }, { "name" : "Central_SouthAmerican", "group" : 2 }, { "name" : "Cajun_Creole", "group" : 2 }, { "name" : "Thai", "group" : 1 }, { "name" : "Scandinavian", "group" : 4 }, { "name" : "Greek", "group" : 3 }, { "name" : "American", "group" : 4 }, { "name" : "African", "group" : 2 }, { "name" : "MiddleEastern", "group" : 2 }, { "name" : "EasternEuropean_Russian", "group" : 4 }, { "name" : "Italian", "group" : 3 }, { "name" : "Irish", "group" : 4 }, { "name" : "Mexican", "group" : 2 }, { "name" : "Chinese", "group" : 1 }, { "name" : "German", "group" : 4 }, { "name" : "Mediterranean", "group" : 3 }, { "name" : "Japanese", "group" : 1 }, { "name" : "Moroccan", "group" : 2 }, { "name" : "Southern_SoulFood", "group" : 4 }, { "name" : "English_Scottish", "group" : 4 }, { "name" : "Asian", "group" : 1 }, { "name" : "Southwestern", "group" : 2 } ] ; 
 var width = 650
height = 500;

var color = d3.scale.category20();

var force = d3.layout.force()
.nodes(d3.values(nodes))
.links(links)
.size([width, height])
.linkDistance(function(d) { return 1000 * d.value + 10; })
.charge(-250)
.on("tick", tick)
.start();

var svg = d3.select("div#cuisine-forcenet").append("svg")
.attr("width", width)
.attr("height", height);

var link = svg.selectAll(".link")
.data(force.links())
.enter().append("line")
.attr("class", "link")
.style("stroke-width", function(d) { return 1/(10*d.value); });

var node = svg.selectAll(".node")
.data(force.nodes())
.enter().append("g")
.attr("class", "node")
.style("fill", function(d) { return color(d.group); })
.style("opacity", 1)
.on("mouseover", mouseover)
.on("mouseout", mouseout)
.call(force.drag);

node.append("circle")
.attr("r", 6)

node.append("svg:text")
.attr("class", "nodetext")
.attr("dx", 12)
.attr("dy", ".35em")
.text(function(d) { return d.name });

function tick() {
link
.attr("x1", function(d) { return d.source.x; })
.attr("y1", function(d) { return d.source.y; })
.attr("x2", function(d) { return d.target.x; })
.attr("y2", function(d) { return d.target.y; });

node.attr("transform", function(d) { return "translate(" + d.x + "," + d.y + ")"; });
}

function mouseover() {
d3.select(this).select("circle").transition()
.duration(750)
.attr("r", 16);
d3.select(this).select("text").transition()
.duration(750)
.attr("x", 13)
.style("stroke-width", ".5px")
.style("font", "17.5px serif")
.style("opacity", 1);
}

function mouseout() {
d3.select(this).select("circle").transition()
.duration(750)
.attr("r", 8);
}

</script>

Feel free to checkout the [source]({{site.github}}/masrab.github.io/) for this post to see all the code for preparing the data for this D3 visualization.
