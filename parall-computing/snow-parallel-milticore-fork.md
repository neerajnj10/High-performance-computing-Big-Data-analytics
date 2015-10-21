---
output: word_document
---
Neeraj-Parallel Computing.
--------------------------------

The `sir.adm` data set has five variables in addition to the patient `id`: `pneu` (pneumonia), `status`, `time`, `age`, and `sex`. See the help file for definitions.

```{r}
library(mvna)
data(sir.adm)
#defining functions.
wrapper <- function(idx) {
    index <- sample(1:nrow(sir.adm), 1000, replace = TRUE)
    temp <- sir.adm[index, ]
    fit <- crr(temp$time, temp$status, temp$pneu)

    return(fit$coef)
}

conf.int <- function(x) {
    n <- length(x)
    m.x <- mean(x)
    sd.x <- sqrt(var(x))
    c.i <- c(m.x - 1.96 * sd.x/sqrt(n), m.x + 1.96 * sd.x/sqrt(n))
    return(c.i)
}

elapsed.time <- matrix(0, 12)
row.names(elapsed.time) <- c("boot", "sock", "sock_lb", "sock_par", "multicore", "multicore_lb", "fork", "fork_lb", "fork_par", "psock", 
    "psock_lb", "psock_lb")
head(sir.adm)
```

Fine and Gray developed a Cox regression approach to model the subdistribution hazard function of a specific cause. The cumulative incidence function is given by:

\[P_1(t; x) = 1 - exp\{-\Lambda_1(t)exp(x'\beta)\}\]
where \(\Lambda_1(t)\) is an increasing function and \(\beta\) is a vector of regression coefficients. This model is implemented in the `crr()` function in the `cmprsk` package.

```{r}
library(cmprsk)
# help(crr)
attach(sir.adm)
##system.time(fit <- crr(time, status, pneu))
tm <- system.time(fit <- lapply(1:1000, wrapper))
elapsed.time[1, ] <- tm[3]
```

The objective is to estimate the coefficient for pneumonia by bootstrapping. Give the estimate and construct a confidence interval on the estimate in each case below. Also provide a histogram of the estimates in each approach. In each case do time testing for the basic fit.

1. Implement the bootstrap model using the `snow` package with 3 workers. Do this using a socket connection. Determine if loadbalancing and/or I/O create a problem.
```{r,echo=FALSE}


##Snow using Socket Connection.

library(snow)
hosts <- c('localhost','localhost','localhost')
cl1 <- makeCluster(hosts, type = "SOCK")
clusterExport(cl1, "sir.adm")
clusterCall(cl1, function() {
    library(cmprsk)
    NULL
})
tm <- snow.time(fit <- clusterApply(cl1, 1:1000, wrapper))
(elapsed.time[2, ] <- tm$elapsed)
plot(tm)
hist(as.numeric(fit), main = "Histogram of bootstrap estimates using SOCK multicore", 
    xlab = "bootstrap estimates")

mean(as.numeric(fit))
conf.int(as.numeric(fit))

rm(fit, tm)

tm <- snow.time(fit <- clusterApplyLB(cl1, 1:1000, wrapper))
(elapsed.time[3, ] <- tm$elapsed)

plot(tm)
hist(as.numeric(fit), main = "Histogram of bootstrap estimates using SOCK(LB) multicore", 
    xlab = "bootstrap estimates")
mean(as.numeric(fit))

conf.int(as.numeric(fit))

rm(fit, tm)

tm <- snow.time(fit <- parLapply(cl1, 1:1000, wrapper))
(elapsed.time[4, ] <- tm$elapsed)

plot(tm)
hist(as.numeric(fit), main = "Histogram of bootstrap estimates using SOCK(par) multicore", 
    xlab = "bootstrap estimates")
conf.int(as.numeric(fit))

rm(fit, tm)

stopCluster(cl1)

```

The bootstrap estimate of the coefficient and the execution times for the SOCK cluster, SOCK cluster with load balancing and SOCK cluster without I/O problem are shown above. Load balancing and solving I/O problem reduces the execution times for running the bootstrap sample estimation of pneumonia coefficient. Load balancing and solving I/O reducing the execution time by almost 3 seconds from the base lapply using 3 clusters.





2. Implement the bootstrap model using the `multicore` package with 3 cores (workers). Explore the load-balancing issue.
```{r,echo=FALSE}

library(multicore)

tm <- system.time(fit <- mclapply(1:1000, wrapper, mc.cores = 3, mc.set.seed = TRUE))
(elapsed.time[5, ] <- tm[3])

hist(as.numeric(fit), main = "Histogram of bootstrap estimates using multicore", xlab = "bootstrap estimates")

mean(as.numeric(fit))
conf.int(as.numeric(fit))
rm(fit, tm)

tm <- system.time(fit <- mclapply(1:1000, wrapper, mc.cores = 3, mc.set.seed = TRUE, 
    mc.preschedule = FALSE))
(elapsed.time[6, ] <- tm[3])

hist(as.numeric(fit), main = "Histogram of bootstrap estimates using multicore(LB)", 
    xlab = "bootstrap estimates")

mean(as.numeric(fit))
conf.int(as.numeric(fit))
rm(fit, tm)

```

The bootstrap estimate of the coefficient and the execution times for the multicore and muticore with load balancing are shown above. After applying load balancing the execution time increased by almost 5 seconds. So load balancing is not preferable for performing the above work using multicore package.

3. Implement the bootstrap model using the `parallel` package by both the `FORK` and `PSOCK` transports using 3 workers.
```{r,echo=FALSE}
library(parallel)
hosts <- c('localhost','localhost','localhost')
cl2 <- makeCluster(name = hosts, 3, type = "FORK")
clusterExport(cl2, "sir.adm")
clusterCall(cl2, function() {
    library(cmprsk)
    NULL
})

clusterSetRNGStream(cl2)

tm <- system.time(fit <- clusterApply(cl2, 1:1000, wrapper))
(elapsed.time[7, ] <- tm[3])
hist(as.numeric(fit), main = "Histogram of bootstrap estimates using FORK", 
    xlab = "bootstrap estimates")
mean(as.numeric(fit))
conf.int(as.numeric(fit))

rm(fit, tm)

tm <- system.time(fit <- clusterApplyLB(cl2, 1:1000, wrapper))
(elapsed.time[8, ] <- tm[3])
hist(as.numeric(fit), main = "Histogram of bootstrap estimates using FORK(LB)", 
    xlab = "bootstrap estimates")

mean(as.numeric(fit))
conf.int(as.numeric(fit))
rm(fit, tm)

tm <- system.time(fit <- parLapply(cl2, 1:1000, wrapper))
(elapsed.time[9, ] <- tm[3])
hist(as.numeric(fit), main = "Histogram of bootstrap estimates using FORK(par)", 
    xlab = "bootstrap estimates")


mean(as.numeric(fit))
conf.int(as.numeric(fit))
rm(fit, tm)



###  The bootstrap estimate of the coefficient and the execution times for the FORK cluster, FORK cluster with load balancing and FORK cluster without I/O problem are shown above. The execution time is reduced by almost 2 seconds when using load balancing and almost 2.5 seconds when solving for I/O problem.

##PSOCK
cl <- makeCluster(hosts, type = "PSOCK")
clusterExport(cl, "sir.adm")
clusterCall(cl, function() {
    library(cmprsk)
    NULL
})
clusterSetRNGStream(cl)

tm <- system.time(fit <- clusterApply(cl, 1:1000, wrapper))
(elapsed.time[10, ] <- tm[3])
hist(as.numeric(fit), main = "Histogram of bootstrap estimates using PSOCK", 
    xlab = "bootstrap estimates")
mean(as.numeric(fit))
conf.int(as.numeric(fit))

rm(fit, tm)

tm <- system.time(fit <- clusterApplyLB(cl, 1:1000, wrapper))
(elapsed.time[11, ] <- tm[3])
hist(as.numeric(fit), main = "Histogram of bootstrap estimates using PSOCK(LB)", 
    xlab = "bootstrap estimates")
mean(as.numeric(fit))
conf.int(as.numeric(fit))
rm(fit, tm)

tm <- system.time(fit <- parLapply(cl, 1:1000, wrapper))
(elapsed.time[12, ] <- tm[3])
hist(as.numeric(fit), main = "Histogram of bootstrap estimates using PSOCK(par)", 
    xlab = "bootstrap estimates")
mean(as.numeric(fit))
conf.int(as.numeric(fit))     
rm(fit, tm)

stopCluster(cl)


###  The bootstrap estimate of the coefficient and the execution times for the PSOCK cluster, PSOCK cluster with load balancing and PSOCK cluster without I/O problem are shown above. Similar result as FORK clusters can be seen above. Almost 2.5 second of reduction in execution time can be seen while using load balancing and also solving for I/O.

```
4. Compare the execution times (adjusted for the number of samples) for these approaches in 1--4 above to each other and to the non-parallel code given above and discuss.
```{r,echo=FALSE}
elapsed.time  # Comparison tothe non-parallel code.
##The bootstrap estimate of the coefficient and the execution times for the snowfall cluster and snowfall cluster with load balancing are shown above. The execution time in both of them are similar.


elapsed.time[-1, ]  # Comparison to each other

## Almost similar execution times can be seen in all the processes. We get the lowest execution time when we used the MPI clusters solving for the I/O problem and maximum for multicore with load balancing.



```




