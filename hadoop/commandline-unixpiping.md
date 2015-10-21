Assignment : Hadoop.
--------------------------------

The data comes from Internet Information Server (IIS) logs for msnbc.com and news-related portions of msn.com for the entire day of September, 28, 1999 (Pacific Standard Time). Each sequence in the dataset corresponds to page views of a user during that twenty-four hour period. Each event in the sequence corresponds to a user's request for a page. Requests are not recorded at the finest level of detail---that is, at the level of URL, but rather, they are recorded at the level of page category (as determined by a site administrator). The categories are "frontpage", "news", "tech", "local", "opinion", "on-air", "misc", "weather", "health", "living", "business", "sports", "summary", "bbs" (bulletin board service), "travel", "msn- news", and "msn-sports". Any page requests served via a caching mechanism were not recorded in the server logs and, hence, not present in the data.

Create two R scripts `mapper.R` and `reducer.R`plus pre- and post-processing R code. The objective is to explore how the relationships among the page categories change over the course of the day. There are 17 page categories.

1. Preprocess (and subset) the data by pushing a 1 into the first position of the first 1,000 observations out of the first 100,000 users, a 2 into the  first position out of the first 1,000 observations from the second 100,000 users, etc. for all 989,818 users. This results in 10 batches of 1,000 each with the batch number in the first position. These 10 values (1 to 10) will become the keys the the `mapper.R` code in the next item and represent a time index throughout the day.

```
msnbc <- readLines("msnbc990928.seq")[-(1:6)]
keys<-c(rep(1,1000),rep(2,1000),rep(3,1000),rep(4,1000),rep(5,1000)
      ,rep(6,1000),rep(7,1000),rep(8,1000),rep(9,1000),rep(10,1000))

values <- msnbc[c(1:1000, 100001:101000, 200001:201000, 300001:301000, 400001:401000, 500001:501000, 600001:601000, 700001:701000, 800001:801000, 900001:901000)]

msnbc <- paste (keys, values, sep="\t")

cat(msnbc, file="msnbc.txt",sep="\n" )
```

Give an overview of your code for `premsnbc.R` here.

* we have done subsetting of the sequence inout file to obtain key value pairs, extracting only 1000 observation for each 100k users. We have used the paste function to make tab separated file for premsnbc-a preprocessor file. keys are the first column while remaining columns are values attached with each of the keys. Keys go as 1 for 1000 times, 2 for next 1000 etc.

2. Develop the code for mapper.R. The key should be 1 for the first 1,000 observations (batch 1), 2 for the second 1,000 observatons (batch 2), etc. The value consists of the counts for each of the 17 page categories for each user-session. Note that the time order for each user-session is lost.

```
#! /usr/bin/env Rscript

input <- file( "stdin" , "r" )

while( TRUE ){
  
  currentLine <- readLines( input , n=1 )
  if( 0 == length( currentLine ) ){
    break
  }

current.value<- rep(0,17)
names(current.value) <- 1:17

current.field<-unlist( strsplit( currentLine, "\t" ) )

split2 <- unlist(strsplit(current.Field[2], " "))

freqtable <- table(split2)

current.value[names(freqtable)] <- freqtable[names(freqtable)]

vresult <- paste(current.value, collapse=",")

result <- paste( current.field[1], vresult, sep="\t" )

cat( result, "\n", append=TRUE)

}

close( input )

```

Give an overview of your code for `mapper.R` and what it produces here.

* first line is used to make readable file for processing at command line. Then we read the input file which is preprocessed file we just created above, one line at a time, until the while loop reaches the end of the file. Then we string split and unlist the input, as it will be obtained in list format. names(current.value) is for 17 different page categories and their respective values for each key. freqtable is used to create the frequency count table. The paste command at the ned is to used to generate the file for each keys ad their associated frequency "count" values. This value produce the mapper output which will go as input to reducer process.

3. Develop the code for reducer.R. Create a `data.frame` for each batch. For each `data.frame`,  change all counts > 1 to 1. Compute the Jaccard distance matrix among the page categories. Note the Jaccard distance is $1 - J$ where $J$ is the Jaccard similarity. Vectorize the result as the return value (the length of the resulting vector will be $17 \times 16/2$). The key is the batch number.   
Hint: Tranpose the `0-1` matrix and multiply it by itself to get a $17 \times 17$ incidence matrix. Compute the Jaccard distanes from this matrix. Then do something like `yourMatrix[lower.tri(yourMatrix)]` to convert to a vector.

```
#! /usr/bin/env Rscript

input <- file( "stdin" , "r" )
lastKey <- ""

tempFile <- tempfile( pattern="hadoop-mr-demo-" , fileext="csv" )
tempHandle <- file( tempFile , "w" )

while( TRUE ){

	currentLine <- readLines( input , n=1 )
	if( 0 == length( currentLine ) ){
		break
	}

	## break this apart into the key and value that were
	## assigned in the Map task
	tuple <- unlist( strsplit( currentLine , "\t" ) )
	currentKey <- tuple[1]
	currentValue <- tuple[2]

	if( ( currentKey != lastKey ) ){
		## a little extra logic here, since the first time through,
		## this conditional will trip

		if( lastKey != "" ){
			## we've hit a new key, so first let's process the
			## data we've accumulated for the previous key:
	
			## close tempFile connection
			close( tempHandle )
	
			## read file of accumulated lines into a data.frame
			x <- read.csv( tempFile , header=FALSE )
	
			
    ## process data.frame and write result to standard output
		  
			X <-ifelse(x == 0,0,1)
      xtx <- t(X) %*% X
      
			dist <- matrix(0,nrow=17, ncol=17)
      
  
      
			for (i in 1:ncol(xtx)) {
			  for (j in i:ncol(xtx)){
			    dist[j,i] <- dist[i,j] <- 1- (xtx[i,j] / (xtx[i,i] + xtx[j,j] - xtx[i,j])) 
			  } 
			}
      
			result <- dist[lower.tri(dist)]
	
			## write result to standard output
			cat(lastKey, paste(result, collapse=","), "\n", append=TRUE)
	
			## cleaup, and start fresh for the next round
			tempHandle <- file( tempFile , "w" )
		}

		lastKey <- currentKey

	}

	# now, either we're still accumulating data for the same key
	# or we have just started a new file.  Either way, we dump a line
	# to the file for later processing.
	cat( currentValue , "\n" , file=tempHandle )

}

## handle the last key, wind-down, cleanup
close( tempHandle )

x <- read.csv( tempFile , header=FALSE )

X <-ifelse(x >= 1,1,0)
xtx <- t(X) %*% X

dist <- matrix(0,nrow=17, ncol=17)

for (i in 1:ncol(xtx)) {
  for (j in i:ncol(xtx)){
    dist[j,i] <- dist[i,j] <- 1 - (xtx[i,j] / (xtx[i,i] + xtx[j,j] - xtx[i,j])) 
  }
  
}

result <- dist[lower.tri(dist)]

cat(currentKey, paste(result, collapse=","), "\n", append=TRUE)

unlink( tempFile )

close( input )


```
* As we have explained above, we are trying to get the final output from the process after recieving mapper out in the form key and the count values. In this process we accumulate all the kay and their resultant fina values, turn them into dataframe, each for ssingle batcch and carry the pprocess to find out the jaccard distance between the categories as our final result.

4. Outside Hadoop, post-process the output from the reducer. For each batch reconstruct the Jaccard distance matrix and convert to a `dist` object, e.g., using `as.dist`. Cluster each of the 10 batches in order using the `average` method in `hclust`. Discuss the changes in the dendrograms over time.

```
dat <- readLines("finalout")

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
* Here we are post processing on the finalout file, we obtained from reducer, which has combined values for each key, total  of 10. For each batch we created dist.obj and converetd it to dist matrix form, whcih we followed with applying "average" hclust distancce formula and then plottin the putput to get the desired result. We finally get the 10 batches of plots, and their heirarchial plot describig the page categories visited and likelihood of visiting one page when some other page has been visited. first page onthe first upper dendogram, is always "msn-news", over time from one batch to another, the denodogram distribution is getting more dense and there are more trees/branches on next the tree than the previous one for different average values. 


5. Develop the workflow for items 1 to 4 above with items 2 and 3 being run within HDFS/ Hadoop using Hadoop streaming based on the R scripts. If you cannot get the code to run with Hadoop then use UNIX pipes, with perhaps some loss in points.

* In preprocessig, we simply obtained the key value subset from the origina file using the code given above, which we followed by applying unix piping (since I could not get it run on hadoop), using formula as. 
mapper----> cat msnbc.txt | ./mapper.R > mapperout
reducer----> cat mapperout | ./reducer.R > reducerout

which we can also do in a single line as 
---> cat msnbc.txt | ./mapper.R | ./reducer.R > finalout


This gives out result and prepares the file for post processig where we finally get our desired result. Our main aim is to find out the page categories vsisited by each user on an average andd the heirarchy followed by them, that is if there is any sequence and repition, or if there are pages that are visited togetherr and viisited often. The dendogram gives us the desired result for each key, each batch what is the pattern followed as seen from the plot as given.


**Note:** Your next assignment will redo this assignment using the `datadr` and `rhipe` packages from Tessera. This assignment uses Hadoop streaming.



