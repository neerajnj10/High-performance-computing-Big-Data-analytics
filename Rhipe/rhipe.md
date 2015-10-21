Assignment : Rhipe
--------------------------------

The data comes from Internet Information Server (IIS) logs for msnbc.com and news-related portions of msn.com for the entire day of September, 28, 1999 (Pacific Standard Time). Each sequence in the dataset corresponds to page views of a user during that twenty-four hour period. Each event in the sequence corresponds to a user's request for a page. Requests are not recorded at the finest level of detail---that is, at the level of URL, but rather, they are recorded at the level of page category (as determined by a site administrator). The categories are "frontpage", "news", "tech", "local", "opinion", "on-air", "misc", "weather", "health", "living", "business", "sports", "summary", "bbs" (bulletin board service), "travel", "msn- news", and "msn-sports". Any page requests served via a caching mechanism were not recorded in the server logs and, hence, not present in the data.

The objective is to explore how the relationships among the page categories change over the course of the day. There are 17 page categories.

1. Using Vagrant, create a session directory called `msnbc` and copy `msnbc.txt` into this directory. Create an `msnbc` directory in HDFS and read the `msnbs.txt` file into this HDFS directory as a text file.

```

Preprocessing the data file-msnbc.txt and calling it "msn.txt"
cd C:\Users\Nj_neeraj\VirtualBox VMs\cdh5mr2-rhipe0.75\install-vagrant-master\cdh5mr2-rhipe0.75
mkdir msn
cd msn

#putting msn.txt in it obtained and downloaded and preprocessed from assign3 zip folder.
#going back to rhipe directory.

cd..

#starting vagrant

vagrant up
vagrant ssh virtualbox

#we are in vagrant now.
Access the localhost:9787 for Rstudio on server to get access to RHIPE(Hadoop in R Server).
vagrant@precise64 opens up.
Then make directory within hadoop from vagrantbox

mkdir msn
cd msn

#Then read and copy msn.txt file present in vagrant in remote/local side under directory msn we just created to msnbc directory.

cp /vagrant/msn/msn.txt .
or 
cp /vagrant/msn/ .  #to copy whole directory, I did this.

#Do ls -l to check if the file transfer is successful.
ls -l

#Now we enter the R studio on the localhost.
#we see 'msn' directory created and 'msn.txt' file copied under the directory.

library(datadr)
library(Rhipe)
rhinint()
rhmkdir("/tmp/msn")
rhls("/tmp")

#put data in Hadoop

rhput("msnbc.txt","/tmp/msn/msn.txt")
rhls("/tmp/msn")
rhexists("/tmp/msn/msn.txt")

#reading the file.

msnbc<-read.table("msn.txt", sep=",")
head(msn)

```

2. Write a `mapper` expression which codes the key be 1 for the first 1,000 observations out of the first 100,000 users (batch 1), 2 for the second 1,000 observatons from the second 100,000 users (batch 2), etc. for all 989,818 users. The `outputkey` consists of the character vector of length of 1 containing the batch number. The `outputvalue` consists of the counts for each of the 17 page categories collected together in a data frame for each user-session.

Note: Since a text file is read in, the `map.keys`  is 1 to 989,818 and these numbers can be used to create the batch `outputkey`. The corresponding `map.values` is the line containing the observational data. The can be used to generate the `outputvalue`, which is a `data.frame` with one row and 17 category variables.

```
msn <- read.table("msn.txt",sep=",")

library(Rhipe)
source.data.file <- "/tmp/msn/msn.txt"

map1 <- expression({
  lapply(seq_along(map.keys),function(r){
    
    line=strsplit(map.values[[r]],",")[[1]]
    outputkey <- line[1]
    
    r <- unlist(strsplit(line[2], " ")) 
    tab <- table(r)   #make frequency table
   
    j <- rep(0,17)     
    names(j) <- 1:17   #set names as character 1 to 17
    j[names(tab)] <- tab[names(tab)]
    result <- paste(j, collapse=",")

    outputvalue <- as.numeric(unlist(strsplit(result,",")))
    stringsAsFactors = FALSE
    rhcollect(outputkey,outputvalue)
  })
})

```

* Map has input key-value pairs, and output key-value pairs. Each pair has an identifier, the key, and numeric-categorical information, the value. The Map R code is applied to each input key-value pair, producing one output key-value pair. Each application of the Map code to a key-value pair is carried out by a mapper, and there are many mappers running in parallel without communication (embarrassingly parallel) until the Map job completes.

RHIPE creates input key-value pair list objects, map.keys and map.values, based on information that it has. Let r be an integer from 1 to the number of input key-value pairs.map.values[[r]] is the value for key map.keys[[r]]. The housing data inputs come from a text file in the HDFS, msn.txt, By RHIPE convention, for a text file, each Map input key is a text file line number, and the corresponding Map input value is the observations in the line, read into R as a single text string.

This Map code is really a for loop with r as the looping variable, but is done by lapply() because it is in general faster than for r in 1:length(map.keys). The loop proceeds through the input keys, specified by the first argument of lapply. The second argument of the above lapply defines the Map expression with the argument r, an index for the Map keys and values. The function strsplit() splits each character-string line input value into the individual observations of the text line. Then we string split and unlist the input, as it will be obtained in list format. names(current.value) is for 17 different page categories and their respective values for each key. freqtable is used to create the frequency count table.

The result, line, is a list of length one whose element is a character vector whose elements are the line observations. Next we turn to the Map output key-value pairs. outputkey for each text line is a character vector.
The RHIPE function rhcollect() forms a Map output key-value pair for each line, and writes the results to the HDFS as a key-value pair list object.

3. Develop the code for the `reducer` expression. The output key-value pairs of `mapper` are the input key-value pairs to `reducer`. Create a `data.frame` for each batch. For each `data.frame`,  change all counts > 1 to 1. Compute the Jaccard distance matrix among the page categories. Note the Jaccard distance is $1 - J$ where $J$ is the Jaccard similarity. Vectorize the result as the return value (the length of the resulting vector will be $17 \times 16/2$). The `reduceoutputkey` is the batch number. The `reduceoutputvalue` is the vectorized Jaccard distances.   

Hint: Tranpose the `0-1` matrix and multiply it by itself to get a $17 \times 17$ incidence matrix. Compute the Jaccard distanes from this matrix. Then do something like `yourMatrix[lower.tri(yourMatrix)]` to convert to a vector.

```
reduce1 <- expression(
  pre={
    reduceoutputvalue <- data.frame()
  },
  reduce = {
    reduceoutputvalue <- rbind(reduceoutputvalue, do.call(rbind, reduce.values))
  },
  post = {
    reduceoutputkey <- reduce.key[1]
    l <- as.matrix(reduceoutputvalue)
    m <-ifelse(l == 0,0,1)
    n <- t(m) %*% m
    
    d <- matrix(0,nrow=17, ncol=17)
    
    for (i in 1:ncol(n)) {
      for (j in i:ncol(n)){
        d[j,i] <- d[i,j] <- 1 - (n[i,j] / (n[i,i] + n[j,j] - n[i,j])) 
      }
    }

    reduceoutputvalue <- d[lower.tri(d)]
    
    rhcollect(reduceoutputkey, reduceoutputvalue)
  }

)
```

* The output key-value pairs of Map are the input key-value pairs to Reduce. The first task of Reduce is to group its input key-value pairs by unique key. The Reduce R code is applied to the key-value pairs of each group by a reducer. The number of groups varies in applications from just one, with a single Reduce output, to many. For multiple groups, the reducers run in parallel, without communication, until the Reduce job completes. 

RHIPE creates two list objects reduce.key and reduce.values. Each element of reduce.key is the key for one group, and the corresponding element of reduce.values has the values for the group to which the Reduce code is applied. Note the Reduce code has a certain structure: expressions pre, reduce, and post.

As we have explained above, we are trying to get the final output from the process after recieving mapper out in the form key and the count values. In this process we accumulate all the key and their resultant fina values, turn them into dataframe, each for ssingle batcch and carry the pprocess to find out the jaccard distance between the categories as our final result.

4. Develop code for `rhwatch()`. The `input` is `msnbc.txt`, which is a text file. The `output` is a sequence file called `byBatch` which is put into the HDFS directory `msnbc`. Specify `readback = TRUE`.  Run `rhwatch()`.

```
mr1 <- rhwatch(
  map      = map1,
  reduce   = reduce1,
  input    = rhfmt("/tmp/msn/msn.txt", type = "text"),
  output   = rhfmt("/tmp/msn/byBatch", type = "sequence"),
  readback = TRUE
)
# read those values back in
a <- rhread("/tmp/msn/byBatch")
```

5. Outside Hadoop, post-process the output from the reducer. For each batch reconstruct the Jaccard distance matrix and convert to a `dist` object, e.g., using `as.dist`. Cluster each of the 10 batches in order using the `average` method in `hclust`. Discuss the changes in the dendrograms over time.

```
# read those values back in and followed by getting back in vagrant.
# even though the fucntions and whole process of working through with RHIPE is correct, I did not get the output that was expected, instead it returned NAN values, and therefore it shouuld be followed by following post-process script like we did in assignment 2.


a <- rhread("/tmp/msn/byBatch")
rhget("/tmp/msn/byBatch", "/vagrant/msn")

dat <- readLines("part-r-00000")

current.field <- unlist(strsplit(dat , "\t" ))

current.field <- c(current.field[1], current.field[3], current.field[4],current.field[5],
                   current.field[6], current.field[7], current.field[8], current.field[9],
                   current.field[10], current.field[2])

dist.mat <- lapply(1:10, function(x){
                 key.value <- unlist(strsplit (current.field[x], " "))
 
                 key.vector <- as.numeric(unlist(strsplit(key.value[2],",")))
  
                 dist.initial <- matrix(0,nrow=17, ncol=17)
                 
                 dist.initial[lower.tri(dist.initial,diag=F)] <- dist.initial[upper.tri(dist.initial,diag=F)] <-  key.vector
  
                 dist.initial
  
               })

dist.obj <- lapply(1:10, function(x){
            as.dist(dist.mat[[x]]) 
          })

output <- lapply(1:10, function(x){
            hclust(dist.obj[[x]], method= "average")
  
})

###making plots
labels <- c("frontpage", "news", "tech", "local", "opinion",
          "on-air","misc", "weather", "health", "living",
          "business", "sports", "summary", "bbs", "travel",
          "msn-news", "msn-sports")
par(mfrow = c(2, 5 ))
plot( output[[1]],labels= labels, main = "Batch 1", xlab= "Pages", sub = " " )
plot( output[[2]], labels= labels, main = "Batch 2" , xlab= "Pages", sub= " ")
plot( output[[3]], labels= labels, main = "Batch 3", xlab= "Pages", sub= " " )
plot( output[[4]], labels= labels, main = "Batch 4", xlab= "Pages", sub= " " )
plot( output[[5]], labels= labels, main = "Batch 5", xlab= "Pages", sub= " " )
plot( output[[6]], labels= labels, main = "Batch 6", xlab= "Pages", sub= " " )
plot( output[[7]], labels= labels, main = "Batch 7" , xlab= "Pages", sub= " ")
plot( output[[8]], labels= labels, main = "Batch 8", xlab= "Pages", sub= " " )
plot( output[[9]], labels= labels, main = "Batch 9", xlab= "Pages", sub= " " )
plot( output[[10]], labels= labels, main = "Batch 10", xlab= "Pages", sub= " " )

```

* Here we are post processing on the part-r-00000 file, we obtained from reducer, which has combined values for each key, total of 10. For each batch we created dist.obj and converetd it to dist matrix form, whcih we followed with applying “average” hclust distancce formula and then plottin the putput to get the desired result. We finally get the 10 batches of plots, and their heirarchial plot describig the page categories visited and likelihood of visiting one page when some other page has been visited. first page onthe first upper dendogram, is always “msn-news”, over time from one batch to another, the denodogram distribution is getting more dense and there are more trees/branches on next the tree than the previous one for different average values.


* We start by preprocessing the data and obtaining key and values file, we give this as an input to rhipe env. ad make it available to map function, followed bby reducer. The rhwatch carries out the rocessing on map and reduce and returns us the result. This gives out result and prepares the file for post processing where we finally get our desired result. Our main aim is to find out the page categories vsisited by each user on an average andd the heirarchy followed by them, that is if there is any sequence and repition, or if there are pages that are visited togetherr and viisited often. The dendogram gives us the desired result for each key, each batch what is the pattern followed as seen from the plot as given.




