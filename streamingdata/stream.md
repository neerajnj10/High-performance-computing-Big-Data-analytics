Assignment : Streams
--------------------------------

Generate bivariate normal streaming data with different means represening three clusters.

The objective of this exercise is to gererate, explore, and evaluate streaming data. You need to explain the code in each item and discuss the output. Refer to the help files for specific functions.

1. Generate bivariate normal streaming data with different means represening three clusters. Explain how and what is generated, i.e., explain `dsd`.



```{r}
library("stream")
set.seed(1002)
dsd <- DSD_Gaussians(k=3, d=2, mu=rbind(c(1.5,1.3),c(1,1),c(1.2,1)))
dsd
```




* Here we first load the "stream" library to start our process. The `stream` framework provides an R-based alternative to `MOA` which seamlessly integrates with the extensive existing R infrastructure. It consists of DSD and DST objects required for streaming purpose. It provides the static, concept drift and connector to real data and streams along with `in-flight` stream operations as well.

* We follow it by setting the seed. They are primarily used to random numbe generation. The need is the possible desire for reproducible results, which may for example come from trying to debug your program, or of course from trying to redo what it does. In essence, these RNGs are called Pseudo Random Number Generators because they are in fact fully algorithmic: given the same seed, you get the same sequence. And that is a feature and not a bug.Once we set the seed, we produce the idential results each time.

* We create a DSD objeyData stream data (DSD) simulates or connects to a data stream. DSD class extend the abstract class `DSD` and a simple interface consisting of 2 functions. A creator function and a data generator function. Here we are using the creator function of the class `DSD_Guassians`. We then specifying the number of clusters as `k = 3` and a data dimensionality of `d = 2` which means our data representation is in 2-Dimension.  Each cluster is represented by a Bivariate Normal Gaussian distribution with a randomly chosen mean (cluster center) in the form of matrix for each dimension of eac cluster, and then follow it by row-binding the resulting individual matrix.


2. Perform a threshold nearest neighbor micro-clustering. Explain what is generated at every step. Do the evaluations measures provide good fits?



```{r}
tnn <- DSC_tNN(r=.1)
tnn
update(tnn, dsd, n=500, verbose=FALSE)
tnn
head(get_centers(tnn))
evaluate(tnn, dsd, measure = c("purity", "crand"), n = 500)
```




* After choosing a `DSD` class to use as the data stream source, our next step isto define a data stream task (`DST`). In `stream`, a `DST` refers to any data mining task that can be applied to data streams, however these are base class shown merely for conceptual purpose and is not directly visible in the code. The reason is that the actual implementations of data stream operators (`DSO`), clustering (`DSC`), classification (`DSClass`) or frequent pattern mining (`DSFPM`) are typically quite different and the benefit of sharing methods would be minimal. Here, we have implemented data stream clustering (`DSC`) since stream currently focuses on this task. So we create DSC, Data stream clustering algorithms, a subclasses of the abstract class `DSC`. DSC_tNN provides interface to Threshold Nearest Neighbor Data Stream Clustering Algorithm as required by us. `r`=0.1 which is the actual threshold in this nearest neighbor algorithm. `Note this process creates the empty cluster.`

* After creating an empty clustering, we are ready to cluster data from the stream using the `update()` function. Here `update()` will implicitly alter the mutable DSC object so no reassignment is necessary. After clustering n= `500` data points, the clustering contains 12 micro-clusters and 2 macro-clusters. Note that the implementation of D-Stream has built-in reclustering and therefore also shows macro-clusters. 

* `head(get_centers(tnn))` returns the first few micro-cluster centers.

* Evaluation of our clustering only measures how well the algorithm learns static structure in the data. Data streams often exhibit concept drift and it is important to evaluate how well the algorithm is able to adapt to these changes. `stream` can be used to evaluate clustering algorithms in terms of learning static structures and clustering dynamic streams. `evaluate` is the function used to perform this task, where `tnn` is the evaluated clustering, `n` is no. of data points fetched from `dsd`, in our case it is n=500. Since we have not mentioned the `assignment` so by default the points are assigned to micro-clusters. Finally, the evaluation `measure` specified in measure is calculated. Several measures can be specified as a vector of character strings.

* We have used `External` measures that use the ground truth (i.e., true partition of the data into groups) to evaluate the agreement of the partition created by the clustering algorithm with a known true partition. The two measures used are `purity` and `crand`. `"purity"` is the Average purity of clusters. The purity of each cluster is the proportion of the points of the majority true group assigned to it. `cRand` is the Rand index corrected for chance (clue).

* verbose is used to report the progress. y default it is True. Assigning verbose=FALSE means we do not expect the progress report.

* We see that Purity of the micro-clusters is high since each micro-cluster only covers points from the same true cluster, however, the corrected Rand index is very low because several micro-clusters split the points from each true cluster.



3. Plot the micro-clusters. Interpret the plot.

```{r}
plot(tnn, dsd)
```




* This plots the `micro-clusters` as shown. We have not performed reclustering hence Macro-clusters are not shown here

* The micro-clusters are plotted in red on top of gray data points. The size of the micro-clusters indicates the weight, i.e., the number of data points represented by each micro-cluster. We see about 5 fairly large micro-clusters in the plot.



4. Perform and plot a $k$-means macro cluster. Explain what is generated at every step. Interpret the plot.

```{r}
kmeans <- DSC_Kmeans(k=3)
recluster(kmeans, tnn)
plot(kmeans, dsd, type = "both")
```




* It is one of the algorithm that is `specifically` used for reclustering of micro-clusterings.

* Here we are submitting/defining the k-means algorithm with no. of clusters as 3. That is, it should result in 3 macro-clusters. Notice weighted is included, which if, is TRUE, then the algorithm gets ignored.

* `recluster` is where we use a macro clustering algorithm defined above, to recluster micro-clusters into a final clustering. we are applying kmeans on `tnn`

* Followed by plotting to see the results. kmeasn is the `DSC` object we created above, to be plotted. `DSD` is the dsd object to plot the input data in the background. `Note` here we chose the type= plot to be `both`, which  means we want to show both micro and macro clusters on the plot.

* As we can see, there are 12 micro clusters and 3 macro-clusters. Micro clusters are denoted by red circles and macro clusters as large blue crosses. Cluster symbol sizes are proportional to the cluster weights. We see that there are `two overlapping micro clusters in the left part` of the plot and the macro-clustering weighs on them. Macro cluster on left most has most weights followed by the cluster on the top-right corner and then folowed by middle cluster.




5. Do the macro evaluations measures provide good fits? Contrast with 2.

```{r}
evaluate(kmeans, dsd, measure = c("purity", "crand"), n = 500)
```




* Here we are performing evaluation on macro clusters now after we did the reclusteringon micro clustering in our previous stage. Evaluation on a macro-clustering model automatically uses the macro-clusters. For evaluation, `n`=500 new data points are requested from the `dsd` object and each is assigned to its nearest micro-cluster. This assignment is translated into macro-cluster assignments that is `kmeans` defined previously and is evaluated using the ground truth provided by the stream generator. In our case, it is again `purity` and `crand` that we used on micro clusters. It is a good idea to use it again to be able to compare any improvement.

* Here we notice that purity of the cluster has `not` increased "significantly", that is it increased from  approximately ~0.91 to ~ 0.95, however we see huge improvement in `crand` index value, where it jumped from merely approx. ~0.25 to around ~0.845 which we could say was expected as the evaluation was done on macro-clusters that reclustered previously splitted points from the true cluster, also because assigning the points rather to the macro-cluster centers splits these points better and therefore decreases the number of incorrectly assigned points as seen from the values.






