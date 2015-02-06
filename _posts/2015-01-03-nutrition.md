---
layout: post
title: Analysing Public Recipe Data in R
permalink: /blog/analysing-public-recipe-data-in-R
comments: True
---

<div class="message">
Note: You can view the source of this blog post on github. Here I'll highlight some of the code.
</div>
We are going to use a publicly available recipe dataset to answer one simple question: **Cuisine from which countries are the most similar?** There are of course various ways to answer this question but here I'll be using a rather simplistic view and focus only on the ingredients used in the recipes attributed to each cuisine type.

## The Dataset
The dataset we are working with is a collection recipes scraped from `@website` where each record contains the type of the cuisine (e.g. American, French, Chinese) and the ingredients that were used in the recipe.

The data stored in flat text files where each line is one record. here's the first two lines of the raw data file:

{% highlight text %}
[1] "Vietnamese\tvinegar\tcilantro\tmint\tolive_oil\tcayenne\tfish\tlime_juice\tshrimp\tlettuce\tcarrot\tgarlic\tbasil\tcucumber\trice\tseed\tshiitake"
[2] "Vietnamese\tonion\tcayenne\tfish\tblack_pepper\tseed\tgarlic"                                                                                     
{% endhighlight %}

Before we can use this data in R, we need to read the data into R and convert it to a data frame. First, we combine all the recipes from the same cuisine type together to create a food corpus.

{% highlight r %}
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

The next step is to create a feature matrix from this corpus. One simple way to create features is to represent each cuisine by the number of times each ingredient has been used in it. This is essentially the standard _bag-of-words_ representation often used in text analytics.


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

The data is currently in what is known as "wide" format (i.e. each feature is represented as one column). To make our lives easier later on (when plotting the data), let's create a "tall" version of the data as well: 


{% highlight r %}
library(tidyr)
# "wide" --> "tall"
food_tall <- food_df %>% gather("ingredient", "count", -cuisine)
{% endhighlight %}

## Looking at the Data
Once we have the feature matrix, we can use that to find the top ingredients used in each type of food?
  

{% highlight r %}
# calculate the ranking of each ingredient for each cuisine type
food_tall <- food_tall %>% group_by(cuisine) %>% 
  mutate(rank = min_rank(desc(count)))
# top N ingredients
food_tall %>% filter(rank<=20) %>% ggplot +
  geom_tile(aes(ingredient, cuisine, fill = rank)) +
  scale_fill_gradient(low="gold", high="red") +
  theme_bw() +
  theme(axis.text.x = element_text(angle = 90, hjust = 1, vjust = .5),
        panel.grid = element_blank())
{% endhighlight %}

![center](/../images/2015-01-03-nutrition/unnamed-chunk-7-1.png) 



{% highlight r %}
# top ingredients
food_tall %>% filter(rank<=2) %>% arrange(cuisine, rank) %>% kable
{% endhighlight %}



|cuisine                 |ingredient | count| rank|
|:-----------------------|:----------|-----:|----:|
|African                 |onion      |    61|    1|
|African                 |olive_oil  |    60|    2|
|American                |butter     |  2219|    1|
|American                |egg        |  1738|    2|
|Asian                   |soy_sauce  |   588|    1|
|Asian                   |ginger     |   576|    2|
|Cajun_Creole            |onion      |   102|    1|
|Cajun_Creole            |cayenne    |    82|    2|
|Central_SouthAmerican   |garlic     |   137|    1|
|Central_SouthAmerican   |onion      |   131|    2|
|Chinese                 |soy_sauce  |   150|    1|
|Chinese                 |ginger     |   140|    2|
|EasternEuropean_Russian |butter     |    88|    1|
|EasternEuropean_Russian |egg        |    74|    2|
|English_Scottish        |butter     |   137|    1|
|English_Scottish        |wheat      |   127|    2|
|French                  |butter     |   488|    1|
|French                  |egg        |   433|    2|
|German                  |butter     |    29|    1|
|German                  |wheat      |    26|    2|
|Greek                   |olive_oil  |   171|    1|
|Greek                   |garlic     |   100|    2|
|Indian                  |cumin      |   160|    1|
|Indian                  |coriander  |   128|    2|
|Irish                   |butter     |    51|    1|
|Irish                   |wheat      |    43|    2|
|Italian                 |olive_oil  |  1124|    1|
|Italian                 |garlic     |   774|    2|
|Japanese                |soy_sauce  |    83|    1|
|Japanese                |rice       |    62|    2|
|Jewish                  |egg        |   190|    1|
|Jewish                  |wheat      |   156|    2|
|Mediterranean           |olive_oil  |   230|    1|
|Mediterranean           |garlic     |   146|    2|
|Mexican                 |cayenne    |   441|    1|
|Mexican                 |onion      |   378|    2|
|MiddleEastern           |olive_oil  |   149|    1|
|MiddleEastern           |garlic     |   116|    2|
|Moroccan                |olive_oil  |   100|    1|
|Moroccan                |cumin      |    75|    2|
|Scandinavian            |butter     |    49|    1|
|Scandinavian            |egg        |    38|    2|
|Southern_SoulFood       |butter     |   200|    1|
|Southern_SoulFood       |wheat      |   168|    2|
|Southwestern            |cayenne    |    88|    1|
|Southwestern            |garlic     |    67|    2|
|Spanish_Portuguese      |olive_oil  |   183|    1|
|Spanish_Portuguese      |garlic     |   167|    2|
|Thai                    |garlic     |    93|    1|
|Thai                    |fish       |    89|    2|
|Vietnamese              |fish       |    51|    1|
|Vietnamese              |garlic     |    47|    2|

If we use the number of unique ingredients used in different cuisine as a measure of _complexity_, we see that `@ABC` is the most complex and `@BAC` has the simplest recipies. 


{% highlight r %}
# Number of unique ingredients
food_tall %>% summarise(unique_ingredients = sum(count>0)) %>% 
  arrange(desc(unique_ingredients)) %>% kable
{% endhighlight %}



|cuisine                 | unique_ingredients|
|:-----------------------|------------------:|
|American                |                324|
|French                  |                260|
|Asian                   |                255|
|Italian                 |                247|
|Mexican                 |                204|
|Southern_SoulFood       |                198|
|English_Scottish        |                188|
|Spanish_Portuguese      |                188|
|Jewish                  |                187|
|Mediterranean           |                184|
|Indian                  |                182|
|MiddleEastern           |                177|
|Central_SouthAmerican   |                169|
|EasternEuropean_Russian |                167|
|Cajun_Creole            |                161|
|African                 |                158|
|Chinese                 |                158|
|Greek                   |                152|
|Moroccan                |                147|
|Southwestern            |                147|
|Scandinavian            |                144|
|Japanese                |                143|
|Thai                    |                138|
|Irish                   |                131|
|German                  |                101|
|Vietnamese              |                101|



{% highlight r %}
# add plot
{% endhighlight %}

## Hierarchical Clustering
We now have everything we need to try and answer the question that motivated our analysis. We want to know which cuisine are the most similar in terms of their igredients. We can use a clustering algorithm and divide our samples into different clusters in an unsupervised fashion. In a problem like this where we don't have a predefined number of clusters in mind, hierarchical clustering is a good option. The algorithm starts by assigning each sample to its own cluster and then proceeds iteratively, at each stage joining the two most similar clusters, continuing until there is just a single cluster.


{% highlight r %}
hc <- data.frame(food_df, row.names = "cuisine")

# without standardizing
hc %>% dist %>% hclust %>% plot(hang= -1, xlab = "")
{% endhighlight %}

![center](/../images/2015-01-03-nutrition/unnamed-chunk-10-1.png) 

Th

We need to be careful when using Euclidean distance to measure similarity. We need to standardize rows prior to calculating dissimilarity:
  

{% highlight r %}
# need to standardize rows prior to calculating dissimilarity
norm_rows <- function(mat){
  mat / rowSums(mat)
}

d <- hc %>% 
  norm_rows %>%
  dist

#hclust(d) %>% plot(hang= -1)

food_dend <- hclust(d) %>% as.dendrogram

num_clusters <- 4
food_dend %>% set("branches_k_color", k = num_clusters) %>% plot

# add bounding boxes
food_dend %>% rect.dendrogram(k=num_clusters, border = 8, lty = 5, lwd = 2)
{% endhighlight %}

![center](/../images/2015-01-03-nutrition/unnamed-chunk-11-1.png) 

## D3 Forced Network
Another way to visualize. Helps us visualize the pairwise similarities between all samples in our data.
<script src="http://d3js.org/d3.v3.min.js" charset="utf-8"></script>
  

{% highlight r %}
create_nodes <- function(d, Group = NULL){
  # d: dissimilarity matrix with row names
  if (is.null(Group)) Group <- 1 # all in the same group
  
  nodes <- data.frame(Name = rownames(d), Group = Group)
  
  nodes
  
}


create_links <- function(d){
  node_names <- rownames(d)
  
  d[upper.tri(d, diag = T)] <- NA
  links <- as.data.frame.table(d, responseName = "Value", stringsAsFactors = FALSE)
  colnames(links)[1:2] <- c("Source", "Target")
  
  links$Source <- factor(links$Source, levels = node_names) %>% as.numeric %>% "-"(1)
  links$Target <- factor(links$Target, levels = node_names) %>% as.numeric %>% "-"(1)
  
  # get rid of self-links and duplicates
  links <- links[!is.na(links$Value),] 
  links
}
{% endhighlight %}

<div id="network1"></div>
  

{% highlight r %}
nodes <- as.matrix(d) %>% 
  create_nodes(Group = cutree(food_dend, k = num_clusters))

links <-  as.matrix(d) %>% 
  create_links

d3ForceNetwork(Links = links, Nodes = nodes, Source = "Source",
               Target = "Target", Value = "Value", NodeID = "Name",
               Group = "Group",
               linkDistance = "function(d) { return 1000 * d.value + 10; }",
               linkWidth = "function(d) { return 1/(10*d.value); }",
               charge = -250,
               opacity = 1, 
               standAlone = FALSE,
               parentElement = "div#network1")
{% endhighlight %}

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

<script src=http://d3js.org/d3.v3.min.js></script>

<script> 
 var links = [ { "source" : 1, "target" : 0, "value" : 0.141309472778934 }, { "source" : 2, "target" : 0, "value" : 0.152263847569019 }, { "source" : 3, "target" : 0, "value" : 0.170116188824581 }, { "source" : 4, "target" : 0, "value" : 0.166581645070816 }, { "source" : 5, "target" : 0, "value" : 0.142586675882655 }, { "source" : 6, "target" : 0, "value" : 0.143216566800016 }, { "source" : 7, "target" : 0, "value" : 0.0757184368151617 }, { "source" : 8, "target" : 0, "value" : 0.177116669830965 }, { "source" : 9, "target" : 0, "value" : 0.178433645436915 }, { "source" : 10, "target" : 0, "value" : 0.149829738622726 }, { "source" : 11, "target" : 0, "value" : 0.146070259534495 }, { "source" : 12, "target" : 0, "value" : 0.157342343619117 }, { "source" : 13, "target" : 0, "value" : 0.176382714785413 }, { "source" : 14, "target" : 0, "value" : 0.170697038938428 }, { "source" : 15, "target" : 0, "value" : 0.189271766416615 }, { "source" : 16, "target" : 0, "value" : 0.14625006285164 }, { "source" : 17, "target" : 0, "value" : 0.137052096792213 }, { "source" : 18, "target" : 0, "value" : 0.182269652681468 }, { "source" : 19, "target" : 0, "value" : 0.167227723758919 }, { "source" : 20, "target" : 0, "value" : 0.133811069351374 }, { "source" : 21, "target" : 0, "value" : 0.165639861591258 }, { "source" : 22, "target" : 0, "value" : 0.167132819464038 }, { "source" : 23, "target" : 0, "value" : 0.200996427134024 }, { "source" : 24, "target" : 0, "value" : 0.094238434136986 }, { "source" : 25, "target" : 0, "value" : 0.145585809437715 }, { "source" : 2, "target" : 1, "value" : 0.138578744890513 }, { "source" : 3, "target" : 1, "value" : 0.144452902927422 }, { "source" : 4, "target" : 1, "value" : 0.151372056515188 }, { "source" : 5, "target" : 1, "value" : 0.119801962527452 }, { "source" : 6, "target" : 1, "value" : 0.132056502657252 }, { "source" : 7, "target" : 1, "value" : 0.101979873387097 }, { "source" : 8, "target" : 1, "value" : 0.149941336691092 }, { "source" : 9, "target" : 1, "value" : 0.160666925147866 }, { "source" : 10, "target" : 1, "value" : 0.129774725371474 }, { "source" : 11, "target" : 1, "value" : 0.0894080317707846 }, { "source" : 12, "target" : 1, "value" : 0.113576340582768 }, { "source" : 13, "target" : 1, "value" : 0.155039043125554 }, { "source" : 14, "target" : 1, "value" : 0.161528845480167 }, { "source" : 15, "target" : 1, "value" : 0.171457932417043 }, { "source" : 16, "target" : 1, "value" : 0.127008937271958 }, { "source" : 17, "target" : 1, "value" : 0.160096721259177 }, { "source" : 18, "target" : 1, "value" : 0.161622942615864 }, { "source" : 19, "target" : 1, "value" : 0.150009334830704 }, { "source" : 20, "target" : 1, "value" : 0.162623287727363 }, { "source" : 21, "target" : 1, "value" : 0.108800326629006 }, { "source" : 22, "target" : 1, "value" : 0.146882182473692 }, { "source" : 23, "target" : 1, "value" : 0.178893618140505 }, { "source" : 24, "target" : 1, "value" : 0.124287029470481 }, { "source" : 25, "target" : 1, "value" : 0.124477510324697 }, { "source" : 3, "target" : 2, "value" : 0.118834766034534 }, { "source" : 4, "target" : 2, "value" : 0.101874484189859 }, { "source" : 5, "target" : 2, "value" : 0.0869989089736921 }, { "source" : 6, "target" : 2, "value" : 0.0912146379693411 }, { "source" : 7, "target" : 2, "value" : 0.148378538617616 }, { "source" : 8, "target" : 2, "value" : 0.139361215360213 }, { "source" : 9, "target" : 2, "value" : 0.0864930881386202 }, { "source" : 10, "target" : 2, "value" : 0.0923212763297632 }, { "source" : 11, "target" : 2, "value" : 0.0845242855558932 }, { "source" : 12, "target" : 2, "value" : 0.0857655176168215 }, { "source" : 13, "target" : 2, "value" : 0.121440188289424 }, { "source" : 14, "target" : 2, "value" : 0.0801718283026827 }, { "source" : 15, "target" : 2, "value" : 0.15377879321885 }, { "source" : 16, "target" : 2, "value" : 0.10998438089799 }, { "source" : 17, "target" : 2, "value" : 0.169098527641798 }, { "source" : 18, "target" : 2, "value" : 0.135797208430996 }, { "source" : 19, "target" : 2, "value" : 0.0637522486691244 }, { "source" : 20, "target" : 2, "value" : 0.17022421353337 }, { "source" : 21, "target" : 2, "value" : 0.100321706184915 }, { "source" : 22, "target" : 2, "value" : 0.120104536911187 }, { "source" : 23, "target" : 2, "value" : 0.157404635433413 }, { "source" : 24, "target" : 2, "value" : 0.142704316334145 }, { "source" : 25, "target" : 2, "value" : 0.107618043416532 }, { "source" : 4, "target" : 3, "value" : 0.075792916452623 }, { "source" : 5, "target" : 3, "value" : 0.129993985878331 }, { "source" : 6, "target" : 3, "value" : 0.120603040949305 }, { "source" : 7, "target" : 3, "value" : 0.164306136585449 }, { "source" : 8, "target" : 3, "value" : 0.0933841146673712 }, { "source" : 9, "target" : 3, "value" : 0.124060560804078 }, { "source" : 10, "target" : 3, "value" : 0.0717932748430411 }, { "source" : 11, "target" : 3, "value" : 0.10910887176174 }, { "source" : 12, "target" : 3, "value" : 0.105992463450293 }, { "source" : 13, "target" : 3, "value" : 0.0723738673628801 }, { "source" : 14, "target" : 3, "value" : 0.117344852844308 }, { "source" : 15, "target" : 3, "value" : 0.0972714587148483 }, { "source" : 16, "target" : 3, "value" : 0.148752123365181 }, { "source" : 17, "target" : 3, "value" : 0.175964575100064 }, { "source" : 18, "target" : 3, "value" : 0.100016120363849 }, { "source" : 19, "target" : 3, "value" : 0.123034252551801 }, { "source" : 20, "target" : 3, "value" : 0.170987179162918 }, { "source" : 21, "target" : 3, "value" : 0.123362333134649 }, { "source" : 22, "target" : 3, "value" : 0.0945932293088356 }, { "source" : 23, "target" : 3, "value" : 0.098764890694245 }, { "source" : 24, "target" : 3, "value" : 0.154756068760404 }, { "source" : 25, "target" : 3, "value" : 0.152196873742273 }, { "source" : 5, "target" : 4, "value" : 0.123162095902027 }, { "source" : 6, "target" : 4, "value" : 0.109949359628071 }, { "source" : 7, "target" : 4, "value" : 0.162248309218897 }, { "source" : 8, "target" : 4, "value" : 0.0833934808002814 }, { "source" : 9, "target" : 4, "value" : 0.111775216476005 }, { "source" : 10, "target" : 4, "value" : 0.0417254348600908 }, { "source" : 11, "target" : 4, "value" : 0.114630965920653 }, { "source" : 12, "target" : 4, "value" : 0.106285992097309 }, { "source" : 13, "target" : 4, "value" : 0.0571997345139504 }, { "source" : 14, "target" : 4, "value" : 0.0883957020795666 }, { "source" : 15, "target" : 4, "value" : 0.0864043606152992 }, { "source" : 16, "target" : 4, "value" : 0.147004176055213 }, { "source" : 17, "target" : 4, "value" : 0.175291442924988 }, { "source" : 18, "target" : 4, "value" : 0.0850751222973209 }, { "source" : 19, "target" : 4, "value" : 0.108089445711319 }, { "source" : 20, "target" : 4, "value" : 0.170996875570309 }, { "source" : 21, "target" : 4, "value" : 0.131066179518986 }, { "source" : 22, "target" : 4, "value" : 0.0757163870683307 }, { "source" : 23, "target" : 4, "value" : 0.0783725046792344 }, { "source" : 24, "target" : 4, "value" : 0.153439764123921 }, { "source" : 25, "target" : 4, "value" : 0.149456329564518 }, { "source" : 6, "target" : 5, "value" : 0.0950078902423833 }, { "source" : 7, "target" : 5, "value" : 0.13614492407241 }, { "source" : 8, "target" : 5, "value" : 0.146288015315305 }, { "source" : 9, "target" : 5, "value" : 0.127732596666761 }, { "source" : 10, "target" : 5, "value" : 0.101967458249754 }, { "source" : 11, "target" : 5, "value" : 0.0937087235839511 }, { "source" : 12, "target" : 5, "value" : 0.105304013489878 }, { "source" : 13, "target" : 5, "value" : 0.127318423238354 }, { "source" : 14, "target" : 5, "value" : 0.118739058675046 }, { "source" : 15, "target" : 5, "value" : 0.155576980473883 }, { "source" : 16, "target" : 5, "value" : 0.0357220266526651 }, { "source" : 17, "target" : 5, "value" : 0.167602544811837 }, { "source" : 18, "target" : 5, "value" : 0.141439467308924 }, { "source" : 19, "target" : 5, "value" : 0.114744210355473 }, { "source" : 20, "target" : 5, "value" : 0.175304580039341 }, { "source" : 21, "target" : 5, "value" : 0.114931825859479 }, { "source" : 22, "target" : 5, "value" : 0.112942846373909 }, { "source" : 23, "target" : 5, "value" : 0.159582093251217 }, { "source" : 24, "target" : 5, "value" : 0.139216675192842 }, { "source" : 25, "target" : 5, "value" : 0.0499112786820937 }, { "source" : 7, "target" : 6, "value" : 0.141130868982129 }, { "source" : 8, "target" : 6, "value" : 0.130329605357684 }, { "source" : 9, "target" : 6, "value" : 0.127618578690261 }, { "source" : 10, "target" : 6, "value" : 0.0936336826378278 }, { "source" : 11, "target" : 6, "value" : 0.105978829595734 }, { "source" : 12, "target" : 6, "value" : 0.113670907419824 }, { "source" : 13, "target" : 6, "value" : 0.11794051075807 }, { "source" : 14, "target" : 6, "value" : 0.118781030327368 }, { "source" : 15, "target" : 6, "value" : 0.139120795417632 }, { "source" : 16, "target" : 6, "value" : 0.110817146339208 }, { "source" : 17, "target" : 6, "value" : 0.154718622284496 }, { "source" : 18, "target" : 6, "value" : 0.126549705256083 }, { "source" : 19, "target" : 6, "value" : 0.119884757049942 }, { "source" : 20, "target" : 6, "value" : 0.156110980334479 }, { "source" : 21, "target" : 6, "value" : 0.131231232385776 }, { "source" : 22, "target" : 6, "value" : 0.0972775480794915 }, { "source" : 23, "target" : 6, "value" : 0.15170470139811 }, { "source" : 24, "target" : 6, "value" : 0.131186645358395 }, { "source" : 25, "target" : 6, "value" : 0.109454753961223 }, { "source" : 8, "target" : 7, "value" : 0.168483502340965 }, { "source" : 9, "target" : 7, "value" : 0.174760416974926 }, { "source" : 10, "target" : 7, "value" : 0.143124276957355 }, { "source" : 11, "target" : 7, "value" : 0.12509720536362 }, { "source" : 12, "target" : 7, "value" : 0.143152982664675 }, { "source" : 13, "target" : 7, "value" : 0.173874871956551 }, { "source" : 14, "target" : 7, "value" : 0.167776115587886 }, { "source" : 15, "target" : 7, "value" : 0.186202006654194 }, { "source" : 16, "target" : 7, "value" : 0.141271417157808 }, { "source" : 17, "target" : 7, "value" : 0.131933555473538 }, { "source" : 18, "target" : 7, "value" : 0.180341976447035 }, { "source" : 19, "target" : 7, "value" : 0.162153263026061 }, { "source" : 20, "target" : 7, "value" : 0.134980440356979 }, { "source" : 21, "target" : 7, "value" : 0.143647218859615 }, { "source" : 22, "target" : 7, "value" : 0.162200245591217 }, { "source" : 23, "target" : 7, "value" : 0.194439442566842 }, { "source" : 24, "target" : 7, "value" : 0.0848167090429891 }, { "source" : 25, "target" : 7, "value" : 0.138171491465505 }, { "source" : 9, "target" : 8, "value" : 0.146815426738202 }, { "source" : 10, "target" : 8, "value" : 0.076297245548567 }, { "source" : 11, "target" : 8, "value" : 0.13763673841861 }, { "source" : 12, "target" : 8, "value" : 0.136505425133527 }, { "source" : 13, "target" : 8, "value" : 0.0720536144903007 }, { "source" : 14, "target" : 8, "value" : 0.139668774657464 }, { "source" : 15, "target" : 8, "value" : 0.091242450206029 }, { "source" : 16, "target" : 8, "value" : 0.166313714735361 }, { "source" : 17, "target" : 8, "value" : 0.180307440639057 }, { "source" : 18, "target" : 8, "value" : 0.0804704246722467 }, { "source" : 19, "target" : 8, "value" : 0.151839860851213 }, { "source" : 20, "target" : 8, "value" : 0.168738165305539 }, { "source" : 21, "target" : 8, "value" : 0.154904289492409 }, { "source" : 22, "target" : 8, "value" : 0.0940559955469568 }, { "source" : 23, "target" : 8, "value" : 0.0901931632162004 }, { "source" : 24, "target" : 8, "value" : 0.157798062363314 }, { "source" : 25, "target" : 8, "value" : 0.166287953914761 }, { "source" : 10, "target" : 9, "value" : 0.10685453723339 }, { "source" : 11, "target" : 9, "value" : 0.108249060023335 }, { "source" : 12, "target" : 9, "value" : 0.0789561077774746 }, { "source" : 13, "target" : 9, "value" : 0.131835521877061 }, { "source" : 14, "target" : 9, "value" : 0.0845121102840735 }, { "source" : 15, "target" : 9, "value" : 0.164024508170234 }, { "source" : 16, "target" : 9, "value" : 0.146246193782798 }, { "source" : 17, "target" : 9, "value" : 0.190887360926453 }, { "source" : 18, "target" : 9, "value" : 0.148363261943351 }, { "source" : 19, "target" : 9, "value" : 0.0521210723698305 }, { "source" : 20, "target" : 9, "value" : 0.185758658247775 }, { "source" : 21, "target" : 9, "value" : 0.111393219915476 }, { "source" : 22, "target" : 9, "value" : 0.139441884190798 }, { "source" : 23, "target" : 9, "value" : 0.163663095915364 }, { "source" : 24, "target" : 9, "value" : 0.167537571640234 }, { "source" : 25, "target" : 9, "value" : 0.145516094384995 }, { "source" : 11, "target" : 10, "value" : 0.0971951436681964 }, { "source" : 12, "target" : 10, "value" : 0.0924438619237694 }, { "source" : 13, "target" : 10, "value" : 0.0591824076824621 }, { "source" : 14, "target" : 10, "value" : 0.087716871761997 }, { "source" : 15, "target" : 10, "value" : 0.0865353131426039 }, { "source" : 16, "target" : 10, "value" : 0.124922488367644 }, { "source" : 17, "target" : 10, "value" : 0.158513240509768 }, { "source" : 18, "target" : 10, "value" : 0.0819139513574867 }, { "source" : 19, "target" : 10, "value" : 0.104197783693116 }, { "source" : 20, "target" : 10, "value" : 0.15435902508506 }, { "source" : 21, "target" : 10, "value" : 0.116734102985087 }, { "source" : 22, "target" : 10, "value" : 0.0584626283306495 }, { "source" : 23, "target" : 10, "value" : 0.0824905874686817 }, { "source" : 24, "target" : 10, "value" : 0.133177294551151 }, { "source" : 25, "target" : 10, "value" : 0.124838187274963 }, { "source" : 12, "target" : 11, "value" : 0.0587527553294329 }, { "source" : 13, "target" : 11, "value" : 0.125600125537742 }, { "source" : 14, "target" : 11, "value" : 0.110779505723334 }, { "source" : 15, "target" : 11, "value" : 0.15333733047502 }, { "source" : 16, "target" : 11, "value" : 0.110196821029092 }, { "source" : 17, "target" : 11, "value" : 0.162583755844035 }, { "source" : 18, "target" : 11, "value" : 0.138792478974773 }, { "source" : 19, "target" : 11, "value" : 0.0915640958996945 }, { "source" : 20, "target" : 11, "value" : 0.165844319606527 }, { "source" : 21, "target" : 11, "value" : 0.0440523943103762 }, { "source" : 22, "target" : 11, "value" : 0.123032625737172 }, { "source" : 23, "target" : 11, "value" : 0.158302603607964 }, { "source" : 24, "target" : 11, "value" : 0.132175640139083 }, { "source" : 25, "target" : 11, "value" : 0.104601850794568 }, { "source" : 13, "target" : 12, "value" : 0.121118823344596 }, { "source" : 14, "target" : 12, "value" : 0.0965958206755896 }, { "source" : 15, "target" : 12, "value" : 0.149453067613152 }, { "source" : 16, "target" : 12, "value" : 0.123957888063344 }, { "source" : 17, "target" : 12, "value" : 0.172595115736224 }, { "source" : 18, "target" : 12, "value" : 0.13848616184639 }, { "source" : 19, "target" : 12, "value" : 0.0698650578335563 }, { "source" : 20, "target" : 12, "value" : 0.170828880084218 }, { "source" : 21, "target" : 12, "value" : 0.0599884885040736 }, { "source" : 22, "target" : 12, "value" : 0.122006710954458 }, { "source" : 23, "target" : 12, "value" : 0.151347872121573 }, { "source" : 24, "target" : 12, "value" : 0.144450270913023 }, { "source" : 25, "target" : 12, "value" : 0.119544818374549 }, { "source" : 14, "target" : 13, "value" : 0.115705855394764 }, { "source" : 15, "target" : 13, "value" : 0.073795297267222 }, { "source" : 16, "target" : 13, "value" : 0.14950724821337 }, { "source" : 17, "target" : 13, "value" : 0.183439311516428 }, { "source" : 18, "target" : 13, "value" : 0.0675017624188165 }, { "source" : 19, "target" : 13, "value" : 0.133115923675707 }, { "source" : 20, "target" : 13, "value" : 0.177512223881702 }, { "source" : 21, "target" : 13, "value" : 0.143499781743017 }, { "source" : 22, "target" : 13, "value" : 0.0740078979842991 }, { "source" : 23, "target" : 13, "value" : 0.0688269888825127 }, { "source" : 24, "target" : 13, "value" : 0.162791790720564 }, { "source" : 25, "target" : 13, "value" : 0.154064630975472 }, { "source" : 15, "target" : 14, "value" : 0.145643536615467 }, { "source" : 16, "target" : 14, "value" : 0.139913076313434 }, { "source" : 17, "target" : 14, "value" : 0.184716159605196 }, { "source" : 18, "target" : 14, "value" : 0.137028535991141 }, { "source" : 19, "target" : 14, "value" : 0.0676105630316842 }, { "source" : 20, "target" : 14, "value" : 0.183965699159022 }, { "source" : 21, "target" : 14, "value" : 0.120625983910468 }, { "source" : 22, "target" : 14, "value" : 0.118415492341247 }, { "source" : 23, "target" : 14, "value" : 0.142672367039376 }, { "source" : 24, "target" : 14, "value" : 0.162464021257847 }, { "source" : 25, "target" : 14, "value" : 0.139615772318461 }, { "source" : 16, "target" : 15, "value" : 0.174359195644683 }, { "source" : 17, "target" : 15, "value" : 0.193737474676764 }, { "source" : 18, "target" : 15, "value" : 0.0899542094762093 }, { "source" : 19, "target" : 15, "value" : 0.163881891917474 }, { "source" : 20, "target" : 15, "value" : 0.185666899853838 }, { "source" : 21, "target" : 15, "value" : 0.169350807911217 }, { "source" : 22, "target" : 15, "value" : 0.0863234943407565 }, { "source" : 23, "target" : 15, "value" : 0.0699313404093615 }, { "source" : 24, "target" : 15, "value" : 0.175026426638776 }, { "source" : 25, "target" : 15, "value" : 0.177037452104563 }, { "source" : 17, "target" : 16, "value" : 0.177945944136915 }, { "source" : 18, "target" : 16, "value" : 0.161745216426086 }, { "source" : 19, "target" : 16, "value" : 0.13346253404928 }, { "source" : 20, "target" : 16, "value" : 0.185917208499186 }, { "source" : 21, "target" : 16, "value" : 0.130074153443747 }, { "source" : 22, "target" : 16, "value" : 0.133839873243633 }, { "source" : 23, "target" : 16, "value" : 0.179924066726895 }, { "source" : 24, "target" : 16, "value" : 0.148888798572139 }, { "source" : 25, "target" : 16, "value" : 0.0414656431388081 }, { "source" : 18, "target" : 17, "value" : 0.18462277776339 }, { "source" : 19, "target" : 17, "value" : 0.185642195778987 }, { "source" : 20, "target" : 17, "value" : 0.0924093707178076 }, { "source" : 21, "target" : 17, "value" : 0.180751930761642 }, { "source" : 22, "target" : 17, "value" : 0.170192169608704 }, { "source" : 23, "target" : 17, "value" : 0.202787430536206 }, { "source" : 24, "target" : 17, "value" : 0.0587153093195449 }, { "source" : 25, "target" : 17, "value" : 0.172984116430093 }, { "source" : 19, "target" : 18, "value" : 0.150759272733677 }, { "source" : 20, "target" : 18, "value" : 0.17916110458296 }, { "source" : 21, "target" : 18, "value" : 0.157044238568918 }, { "source" : 22, "target" : 18, "value" : 0.0873710924942921 }, { "source" : 23, "target" : 18, "value" : 0.0959537289863271 }, { "source" : 24, "target" : 18, "value" : 0.166030321897805 }, { "source" : 25, "target" : 18, "value" : 0.165979839056946 }, { "source" : 20, "target" : 19, "value" : 0.182487514659679 }, { "source" : 21, "target" : 19, "value" : 0.0939156814729362 }, { "source" : 22, "target" : 19, "value" : 0.138816173166382 }, { "source" : 23, "target" : 19, "value" : 0.166390472644856 }, { "source" : 24, "target" : 19, "value" : 0.159681424154718 }, { "source" : 25, "target" : 19, "value" : 0.131841101500615 }, { "source" : 21, "target" : 20, "value" : 0.181924362420775 }, { "source" : 22, "target" : 20, "value" : 0.170238432462396 }, { "source" : 23, "target" : 20, "value" : 0.195273603330744 }, { "source" : 24, "target" : 20, "value" : 0.0747173195715422 }, { "source" : 25, "target" : 20, "value" : 0.182268686910723 }, { "source" : 22, "target" : 21, "value" : 0.144235249528795 }, { "source" : 23, "target" : 21, "value" : 0.172168460087227 }, { "source" : 24, "target" : 21, "value" : 0.152303809871262 }, { "source" : 25, "target" : 21, "value" : 0.123658114127637 }, { "source" : 23, "target" : 22, "value" : 0.0835889717683904 }, { "source" : 24, "target" : 22, "value" : 0.150282375178348 }, { "source" : 25, "target" : 22, "value" : 0.132994805963444 }, { "source" : 24, "target" : 23, "value" : 0.184635303842036 }, { "source" : 25, "target" : 23, "value" : 0.184104855364108 }, { "source" : 25, "target" : 24, "value" : 0.14405258237824 } ] ; 
 var nodes = [ { "name" : "Vietnamese", "group" : 1 }, { "name" : "Indian", "group" : 2 }, { "name" : "Spanish_Portuguese", "group" : 3 }, { "name" : "Jewish", "group" : 4 }, { "name" : "French", "group" : 4 }, { "name" : "Central_SouthAmerican", "group" : 2 }, { "name" : "Cajun_Creole", "group" : 2 }, { "name" : "Thai", "group" : 1 }, { "name" : "Scandinavian", "group" : 4 }, { "name" : "Greek", "group" : 3 }, { "name" : "American", "group" : 4 }, { "name" : "African", "group" : 2 }, { "name" : "MiddleEastern", "group" : 2 }, { "name" : "EasternEuropean_Russian", "group" : 4 }, { "name" : "Italian", "group" : 3 }, { "name" : "Irish", "group" : 4 }, { "name" : "Mexican", "group" : 2 }, { "name" : "Chinese", "group" : 1 }, { "name" : "German", "group" : 4 }, { "name" : "Mediterranean", "group" : 3 }, { "name" : "Japanese", "group" : 1 }, { "name" : "Moroccan", "group" : 2 }, { "name" : "Southern_SoulFood", "group" : 4 }, { "name" : "English_Scottish", "group" : 4 }, { "name" : "Asian", "group" : 1 }, { "name" : "Southwestern", "group" : 2 } ] ; 
 var width = 900
height = 600;

var color = d3.scale.category20();

var force = d3.layout.force()
.nodes(d3.values(nodes))
.links(links)
.size([width, height])
.linkDistance(function(d) { return 1000 * d.value + 10; })
.charge(-250)
.on("tick", tick)
.start();

var svg = d3.select("div#network1").append("svg")
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


Not very clear. What if we prune?

{% highlight r %}
links <- links %>% filter(Value < quantile(d,.4))
{% endhighlight %}

We now get this:
  
  <div id="network2"></div>
  
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

<script src=http://d3js.org/d3.v3.min.js></script>

<script> 
 var links = [ { "source" : 7, "target" : 0, "value" : 0.0757184368151617 }, { "source" : 24, "target" : 0, "value" : 0.094238434136986 }, { "source" : 5, "target" : 1, "value" : 0.119801962527452 }, { "source" : 7, "target" : 1, "value" : 0.101979873387097 }, { "source" : 11, "target" : 1, "value" : 0.0894080317707846 }, { "source" : 12, "target" : 1, "value" : 0.113576340582768 }, { "source" : 21, "target" : 1, "value" : 0.108800326629006 }, { "source" : 24, "target" : 1, "value" : 0.124287029470481 }, { "source" : 25, "target" : 1, "value" : 0.124477510324697 }, { "source" : 3, "target" : 2, "value" : 0.118834766034534 }, { "source" : 4, "target" : 2, "value" : 0.101874484189859 }, { "source" : 5, "target" : 2, "value" : 0.0869989089736921 }, { "source" : 6, "target" : 2, "value" : 0.0912146379693411 }, { "source" : 9, "target" : 2, "value" : 0.0864930881386202 }, { "source" : 10, "target" : 2, "value" : 0.0923212763297632 }, { "source" : 11, "target" : 2, "value" : 0.0845242855558932 }, { "source" : 12, "target" : 2, "value" : 0.0857655176168215 }, { "source" : 13, "target" : 2, "value" : 0.121440188289424 }, { "source" : 14, "target" : 2, "value" : 0.0801718283026827 }, { "source" : 16, "target" : 2, "value" : 0.10998438089799 }, { "source" : 19, "target" : 2, "value" : 0.0637522486691244 }, { "source" : 21, "target" : 2, "value" : 0.100321706184915 }, { "source" : 22, "target" : 2, "value" : 0.120104536911187 }, { "source" : 25, "target" : 2, "value" : 0.107618043416532 }, { "source" : 4, "target" : 3, "value" : 0.075792916452623 }, { "source" : 6, "target" : 3, "value" : 0.120603040949305 }, { "source" : 8, "target" : 3, "value" : 0.0933841146673712 }, { "source" : 9, "target" : 3, "value" : 0.124060560804078 }, { "source" : 10, "target" : 3, "value" : 0.0717932748430411 }, { "source" : 11, "target" : 3, "value" : 0.10910887176174 }, { "source" : 12, "target" : 3, "value" : 0.105992463450293 }, { "source" : 13, "target" : 3, "value" : 0.0723738673628801 }, { "source" : 14, "target" : 3, "value" : 0.117344852844308 }, { "source" : 15, "target" : 3, "value" : 0.0972714587148483 }, { "source" : 18, "target" : 3, "value" : 0.100016120363849 }, { "source" : 19, "target" : 3, "value" : 0.123034252551801 }, { "source" : 21, "target" : 3, "value" : 0.123362333134649 }, { "source" : 22, "target" : 3, "value" : 0.0945932293088356 }, { "source" : 23, "target" : 3, "value" : 0.098764890694245 }, { "source" : 5, "target" : 4, "value" : 0.123162095902027 }, { "source" : 6, "target" : 4, "value" : 0.109949359628071 }, { "source" : 8, "target" : 4, "value" : 0.0833934808002814 }, { "source" : 9, "target" : 4, "value" : 0.111775216476005 }, { "source" : 10, "target" : 4, "value" : 0.0417254348600908 }, { "source" : 11, "target" : 4, "value" : 0.114630965920653 }, { "source" : 12, "target" : 4, "value" : 0.106285992097309 }, { "source" : 13, "target" : 4, "value" : 0.0571997345139504 }, { "source" : 14, "target" : 4, "value" : 0.0883957020795666 }, { "source" : 15, "target" : 4, "value" : 0.0864043606152992 }, { "source" : 18, "target" : 4, "value" : 0.0850751222973209 }, { "source" : 19, "target" : 4, "value" : 0.108089445711319 }, { "source" : 22, "target" : 4, "value" : 0.0757163870683307 }, { "source" : 23, "target" : 4, "value" : 0.0783725046792344 }, { "source" : 6, "target" : 5, "value" : 0.0950078902423833 }, { "source" : 10, "target" : 5, "value" : 0.101967458249754 }, { "source" : 11, "target" : 5, "value" : 0.0937087235839511 }, { "source" : 12, "target" : 5, "value" : 0.105304013489878 }, { "source" : 14, "target" : 5, "value" : 0.118739058675046 }, { "source" : 16, "target" : 5, "value" : 0.0357220266526651 }, { "source" : 19, "target" : 5, "value" : 0.114744210355473 }, { "source" : 21, "target" : 5, "value" : 0.114931825859479 }, { "source" : 22, "target" : 5, "value" : 0.112942846373909 }, { "source" : 25, "target" : 5, "value" : 0.0499112786820937 }, { "source" : 10, "target" : 6, "value" : 0.0936336826378278 }, { "source" : 11, "target" : 6, "value" : 0.105978829595734 }, { "source" : 12, "target" : 6, "value" : 0.113670907419824 }, { "source" : 13, "target" : 6, "value" : 0.11794051075807 }, { "source" : 14, "target" : 6, "value" : 0.118781030327368 }, { "source" : 16, "target" : 6, "value" : 0.110817146339208 }, { "source" : 19, "target" : 6, "value" : 0.119884757049942 }, { "source" : 22, "target" : 6, "value" : 0.0972775480794915 }, { "source" : 25, "target" : 6, "value" : 0.109454753961223 }, { "source" : 24, "target" : 7, "value" : 0.0848167090429891 }, { "source" : 10, "target" : 8, "value" : 0.076297245548567 }, { "source" : 13, "target" : 8, "value" : 0.0720536144903007 }, { "source" : 15, "target" : 8, "value" : 0.091242450206029 }, { "source" : 18, "target" : 8, "value" : 0.0804704246722467 }, { "source" : 22, "target" : 8, "value" : 0.0940559955469568 }, { "source" : 23, "target" : 8, "value" : 0.0901931632162004 }, { "source" : 10, "target" : 9, "value" : 0.10685453723339 }, { "source" : 11, "target" : 9, "value" : 0.108249060023335 }, { "source" : 12, "target" : 9, "value" : 0.0789561077774746 }, { "source" : 14, "target" : 9, "value" : 0.0845121102840735 }, { "source" : 19, "target" : 9, "value" : 0.0521210723698305 }, { "source" : 21, "target" : 9, "value" : 0.111393219915476 }, { "source" : 11, "target" : 10, "value" : 0.0971951436681964 }, { "source" : 12, "target" : 10, "value" : 0.0924438619237694 }, { "source" : 13, "target" : 10, "value" : 0.0591824076824621 }, { "source" : 14, "target" : 10, "value" : 0.087716871761997 }, { "source" : 15, "target" : 10, "value" : 0.0865353131426039 }, { "source" : 18, "target" : 10, "value" : 0.0819139513574867 }, { "source" : 19, "target" : 10, "value" : 0.104197783693116 }, { "source" : 21, "target" : 10, "value" : 0.116734102985087 }, { "source" : 22, "target" : 10, "value" : 0.0584626283306495 }, { "source" : 23, "target" : 10, "value" : 0.0824905874686817 }, { "source" : 25, "target" : 10, "value" : 0.124838187274963 }, { "source" : 12, "target" : 11, "value" : 0.0587527553294329 }, { "source" : 14, "target" : 11, "value" : 0.110779505723334 }, { "source" : 16, "target" : 11, "value" : 0.110196821029092 }, { "source" : 19, "target" : 11, "value" : 0.0915640958996945 }, { "source" : 21, "target" : 11, "value" : 0.0440523943103762 }, { "source" : 22, "target" : 11, "value" : 0.123032625737172 }, { "source" : 25, "target" : 11, "value" : 0.104601850794568 }, { "source" : 13, "target" : 12, "value" : 0.121118823344596 }, { "source" : 14, "target" : 12, "value" : 0.0965958206755896 }, { "source" : 16, "target" : 12, "value" : 0.123957888063344 }, { "source" : 19, "target" : 12, "value" : 0.0698650578335563 }, { "source" : 21, "target" : 12, "value" : 0.0599884885040736 }, { "source" : 22, "target" : 12, "value" : 0.122006710954458 }, { "source" : 25, "target" : 12, "value" : 0.119544818374549 }, { "source" : 14, "target" : 13, "value" : 0.115705855394764 }, { "source" : 15, "target" : 13, "value" : 0.073795297267222 }, { "source" : 18, "target" : 13, "value" : 0.0675017624188165 }, { "source" : 22, "target" : 13, "value" : 0.0740078979842991 }, { "source" : 23, "target" : 13, "value" : 0.0688269888825127 }, { "source" : 19, "target" : 14, "value" : 0.0676105630316842 }, { "source" : 21, "target" : 14, "value" : 0.120625983910468 }, { "source" : 22, "target" : 14, "value" : 0.118415492341247 }, { "source" : 18, "target" : 15, "value" : 0.0899542094762093 }, { "source" : 22, "target" : 15, "value" : 0.0863234943407565 }, { "source" : 23, "target" : 15, "value" : 0.0699313404093615 }, { "source" : 25, "target" : 16, "value" : 0.0414656431388081 }, { "source" : 20, "target" : 17, "value" : 0.0924093707178076 }, { "source" : 24, "target" : 17, "value" : 0.0587153093195449 }, { "source" : 22, "target" : 18, "value" : 0.0873710924942921 }, { "source" : 23, "target" : 18, "value" : 0.0959537289863271 }, { "source" : 21, "target" : 19, "value" : 0.0939156814729362 }, { "source" : 24, "target" : 20, "value" : 0.0747173195715422 }, { "source" : 25, "target" : 21, "value" : 0.123658114127637 }, { "source" : 23, "target" : 22, "value" : 0.0835889717683904 } ] ; 
 var nodes = [ { "name" : "Vietnamese", "group" : 1 }, { "name" : "Indian", "group" : 2 }, { "name" : "Spanish_Portuguese", "group" : 3 }, { "name" : "Jewish", "group" : 4 }, { "name" : "French", "group" : 4 }, { "name" : "Central_SouthAmerican", "group" : 2 }, { "name" : "Cajun_Creole", "group" : 2 }, { "name" : "Thai", "group" : 1 }, { "name" : "Scandinavian", "group" : 4 }, { "name" : "Greek", "group" : 3 }, { "name" : "American", "group" : 4 }, { "name" : "African", "group" : 2 }, { "name" : "MiddleEastern", "group" : 2 }, { "name" : "EasternEuropean_Russian", "group" : 4 }, { "name" : "Italian", "group" : 3 }, { "name" : "Irish", "group" : 4 }, { "name" : "Mexican", "group" : 2 }, { "name" : "Chinese", "group" : 1 }, { "name" : "German", "group" : 4 }, { "name" : "Mediterranean", "group" : 3 }, { "name" : "Japanese", "group" : 1 }, { "name" : "Moroccan", "group" : 2 }, { "name" : "Southern_SoulFood", "group" : 4 }, { "name" : "English_Scottish", "group" : 4 }, { "name" : "Asian", "group" : 1 }, { "name" : "Southwestern", "group" : 2 } ] ; 
 var width = 900
height = 600;

var color = d3.scale.category20();

var force = d3.layout.force()
.nodes(d3.values(nodes))
.links(links)
.size([width, height])
.linkDistance(function(d) { return 1000 * d.value + 10; })
.charge(-250)
.on("tick", tick)
.start();

var svg = d3.select("div#network2").append("svg")
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


