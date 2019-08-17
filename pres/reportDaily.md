---
title: "Time Series Project: Analysis of PM2.5 in Beijing"
author: "David Josephs"
date: "2019-08-14"
output: 
  html_document:
    toc: TRUE
    number_sections: true
    toc_float:
      collapsed: true
      smooth_scroll: false
    theme: paper
    df_print: paged
    keep_md: TRUE

---




# Project Setting

TODO, one day ahead hourly forecasts of PM 2.5


# Data Pre-processing

## Data Definition

Open to the public is data from 5 cities:

* Beijing

* Chengdu

* Guangzhou

* Shanghai

* Shenyang

Each of these datasets contains:

* Air quality measurements at multiple locations (including a US Embassy post)

* Humity

* Air pressure

* Temperature

* Precipitation

* Wind direction

* Wind Speed

Measured at every hour. For detailed info on the data quality and collection methods, please refer to [this publication](https://agupubs.onlinelibrary.wiley.com/doi/full/10.1002/2016JD024877)


## Pre-processing pipeline

The next step in our analysis workflow is to get the data ready for analysis. First, we will have to do a few tasks

* import data

* Represent data as a time series

* inspect for NAs

* figure out how to deal with NAs

* store in a conveniently manipulatable format

All of this can be seen in `R/preprocessing.R`, but we will do an in depth explanation here

### Required libraries


```r
library(functional)  # to compose the preprocessing pipeline
library(data.table)  # to read csvs
library(rlist)  # for list manipulations
library(pipeR)  # fast, dumb pipes
library(imputeTS)  # to impute NAs
library(pander)  # so i can read the output
library(foreach)  # go fast
library(doParallel)  # go fast
```

### Data loading

We want to load all of the csv files stored in `data/`, so we will write a function which loads all of them into a list of `data.table`(fast data frame) objects. We will want the name of each item in the list to be the name of the csv file, minus the useless numbers


```r
import <- function(path) {
    # first we list the files in our path, in this case 'data/'
    files <- list.files(path)
    # then we search for csvs
    files <- files[grepl(files, pattern = ".csv")]
    # paste the csv names onto the filepath(data/whatever.csv)
    filepaths <- sapply(files, function(x) paste0(path, x))
    # read them in to a list
    out <- lapply(filepaths, fread)
    # Get rid of .csv in each filename
    fnames <- gsub(".csv", "", files)
    # get rid of the confusing numbers
    fnames <- gsub("[[:digit:]]+", "", fnames)
    # set the names of the list to be the clean filenams
    names(out) <- fnames
    out
}
```

### NA identification

Next we want to see how many NAs we are dealing with. Since we have about 20 million total observations, we will represent these as a proportion. We will apply the NA count function to our list using the beautiful `rapply` function


```r
naCount <- function(xs) {
    rapply(xs, function(x) sum(is.na(x)/length(x)), how = "list")
}
# load in the data now
datadir = "data/"
china <- import(datadir)
pander(naCount(china))
```



  * **BeijingPM_**:

      * **No**: _0_
      * **year**: _0_
      * **month**: _0_
      * **day**: _0_
      * **hour**: _0_
      * **season**: _0_
      * **PM_Dongsi**: _0.5236_
      * **PM_Dongsihuan**: _0.61_
      * **PM_Nongzhanguan**: _0.5259_
      * **PM_US Post**: _0.04178_
      * **DEWP**: _0.00009509_
      * **HUMI**: _0.006447_
      * **PRES**: _0.006447_
      * **TEMP**: _0.00009509_
      * **cbwd**: _0.00009509_
      * **Iws**: _0.00009509_
      * **precipitation**: _0.009204_
      * **Iprec**: _0.009204_

  * **ChengduPM_**:

      * **No**: _0_
      * **year**: _0_
      * **month**: _0_
      * **day**: _0_
      * **hour**: _0_
      * **season**: _0_
      * **PM_Caotangsi**: _0.5356_
      * **PM_Shahepu**: _0.5323_
      * **PM_US Post**: _0.4504_
      * **DEWP**: _0.01006_
      * **HUMI**: _0.01017_
      * **PRES**: _0.009908_
      * **TEMP**: _0.01002_
      * **cbwd**: _0.009908_
      * **Iws**: _0.01014_
      * **precipitation**: _0.0562_
      * **Iprec**: _0.0562_

  * **GuangzhouPM_**:

      * **No**: _0_
      * **year**: _0_
      * **month**: _0_
      * **day**: _0_
      * **hour**: _0_
      * **season**: _0.00001902_
      * **PM_City Station**: _0.3848_
      * **PM_5th Middle School**: _0.5988_
      * **PM_US Post**: _0.3848_
      * **DEWP**: _0.00001902_
      * **HUMI**: _0.00001902_
      * **PRES**: _0.00001902_
      * **TEMP**: _0.00001902_
      * **cbwd**: _0.00001902_
      * **Iws**: _0.00001902_
      * **precipitation**: _0.00001902_
      * **Iprec**: _0.00001902_

  * **ShanghaiPM_**:

      * **No**: _0_
      * **year**: _0_
      * **month**: _0_
      * **day**: _0_
      * **hour**: _0_
      * **season**: _0_
      * **PM_Jingan**: _0.5303_
      * **PM_US Post**: _0.3527_
      * **PM_Xuhui**: _0.521_
      * **DEWP**: _0.0002472_
      * **HUMI**: _0.0002472_
      * **PRES**: _0.0005325_
      * **TEMP**: _0.0002472_
      * **cbwd**: _0.0002282_
      * **Iws**: _0.0002282_
      * **precipitation**: _0.07624_
      * **Iprec**: _0.07624_

  * **ShenyangPM_**:

      * **No**: _0_
      * **year**: _0_
      * **month**: _0_
      * **day**: _0_
      * **hour**: _0_
      * **season**: _0_
      * **PM_Taiyuanjie**: _0.5362_
      * **PM_US Post**: _0.5877_
      * **PM_Xiaoheyan**: _0.5317_
      * **DEWP**: _0.01316_
      * **HUMI**: _0.01293_
      * **PRES**: _0.01316_
      * **TEMP**: _0.01316_
      * **cbwd**: _0.01316_
      * **Iws**: _0.01316_
      * **precipitation**: _0.2427_
      * **Iprec**: _0.2427_


<!-- end of list -->

It looks like the quality of the data collected outside of the US Embassy posts and in cities other than beijing is really bad. Luckily for us, `BeijingPM$PM_US.Post` has relatively high quality data. That being said, we are still going to have to impute the NAs. Before that though, lets convert this to time series data, as we will be imputing the NAs in a different way with time series data than we will for normal data.

### Data transformation: Hourly time series

First, we define a function which converts the data to a time series object:


```r
tots <- function(v) {
    ts(v, frequency = 365 * 24)
}
```

Next, we are going to want want to scale this up, to work on a whole data frame. We will also convert the data frame to a list, just to avoid any typing issues down the road. We also do not want to convert non time series data to time series, so we must avoid those as well:


```r
totslist <- function(df) {
    # define the column names which we do not want to alter
    badlist <- c("No", "year", "month", "day", "hour", "season", "cbwd")
    # get the column names of the original data frame
    nms <- colnames(df)
    # coerce it into a list
    df <- as.list(df)
    # if the name of the item of the list is in our to not change list, do
    # nothing otherwise, convert to ts
    for (name in nms) {
        if (name %in% badlist) {
            df[[name]] <- df[[name]]
        } else {
            df[[name]] <- tots(df[[name]])
        }
    }
    # return the created list
    df
}
```

Next, we want to scale this function to work on a list of data frames:


```r
totsall <- function(xs) {
    lapply(xs, totslist)
}
```

### NA imputation

Imputing NAs in time series is a tricky topic, as we cannot just ignore them (that would mess up our sampling rate). There are several methods of dealing with NAs in time series, including kalman filters, polynomial interpretation, and spline interpolation. In this case, as we are only concerned with the time series that have relatively few NAs, I elected to use spline interpolation, as it is much faster than the kalman filter, while still providing pretty good accuracy. First, a function was constructed to use the try catch pattern to impute NAs of items with the `time-series` class.

The logic is as follows:

* try spline interpolation on the vector
  * if the output is a `ts` object, return the results
  * if the output is an error or a non time series object, return the original object


```r
imp_test <- function(v) {
    out <- try(na.interpolation(v, "spline"))
    ifelse(is.ts(out), return(out), return(v))
}
```

Next, we apply that function to a single list, in parallel to save time:


```r
impute <- function(xs) {
    foreach(i = 1:length(xs), .final = function(x) {
        setNames(x, names(xs))
    }) %dopar% imp_test(xs[[i]])
}
```

Finally, we apply that function to our list of lists


```r
impute_list <- function(xs) {
    lapply(xs, impute)
}
```

Next we are ready to convert to a more useful data structure, and then pipeline all these functions together for quick preprocessing

### Pipelining

First, we will convert our slow, massive list into the fast, superior hash table. For large, complex datasets, a hash table was shown to be at more or less 3 times faster than a list, and as we will be repeatedly accessing data from this, we would prefer it to be a quick index. For more info on R hash tables, please refer to [this link](https://blog.dominodatalab.com/a-quick-benchmark-of-hashtable-implementations-in-r/).


```r
to_hash <- function(xs) {
    list2env(xs, envir = NULL, hash = TRUE)
}
```

Next we are ready to pipeline it all together into a single preprocessing function, using `functional::Compose`


```r
preprocess <- Compose(import, totsall, impute_list, to_hash)
```


# Univariate EDA

Required libraries


```r
library(ggthemes)
library(ggplot2)
library(cowplot)
```

## Helper functions

First, we define some helper functions, in order to save us time and make our code more readable. First, lets steal some stuff from `forecast` and `tswge`:


```r
# pretty plots for our analysis
seasonplot <- forecast::ggseasonplot
subseriesplot <- forecast::ggsubseriesplot
lagplot <- forecast::gglagplot
sampplot <- tswge::plotts.sample.wge
# clean up errant data points
clean <- forecast::tsclean
```

Next let us define a function which creates a function to resample time series, and then define ones with new sampling rates. Finally, we will write a function which applies each resampling function to a vector


```r
# resample generator
change_samples <- function(n) {
    function(xs) {
        out <- unname(tapply(xs, (seq_along(xs) - 1)%/%n, sum))
        out <- ts(out, frequency = (8760/n))
        out
    }
}

# daily and weekly sampling, monthly is 4 weeks
to_daily <- change_samples(24)
to_weekly <- change_samples(24 * 7)
to_monthly <- change_samples(24 * 7 * 4)
to_season <- change_samples(24 * (365/4))

# pipelining final cleaning and conversion, removing outlier and the couple
# of errant negative values (which do not make sense)
cleandays <- function(xs) {
    xs %>>% clean %>>% abs %>>% to_daily
}

cleanweeks <- function(xs) {
    xs %>>% clean %>>% abs %>>% to_weekly
}
cleanmonths <- function(xs) {
    xs %>>% clean %>>% abs %>>% to_monthly
}
cleanseas <- function(xs) {
    xs %>>% clean %>>% abs %>>% to_season
}

# resample the whole ts
resample <- function(xs) {
    day <- xs %>>% cleandays %>>% window(end = 6)
    week <- xs %>>% cleanweeks %>>% window(end = 6)
    month <- xs %>>% cleanmonths %>>% window(end = 6)
    seas <- xs %>>% cleanseas %>>% window(end = 6)
    list(day = day, week = week, month = month, season = seas)
}
```

## Wge sample plots

First lets load in and resample our data:


```r
library(ggthemes)
library(ggplot2)
library(cowplot)
china <- preprocess(datadir)
```

```
#> Error in na.interpolation(v, "spline") : Input x is not numeric
#> Error in na.interpolation(v, "spline") : Input x is not numeric
#> Error in na.interpolation(v, "spline") : Input x is not numeric
#> Error in na.interpolation(v, "spline") : Input x is not numeric
#> Error in na.interpolation(v, "spline") : Input x is not numeric
```

```r
bj <- china$BeijingPM_$PM_US
# this takes a long time, and we are using it only once so we wil just load
# it like this bj <- resample(bj) save(bj, file = 'resampled.Rda')
load("pres/resampled.Rda")
```

As looking at the sample plots of the hourly data(even the daily data) does not provide much utility, lets look at the sample plots for the daily, weekly, and quarterly data first:

### Daily data

Now lets look at the sample plots, first with the daily data


```r
dy <- sampplot(bj$day)
```

<img src="fig/unnamed-chunk-3-1.png" style="display: block; margin: auto;" />

So looking at this, we see that we have what looks like a wandering behavior, according to the frequency plot, but the lags tell a different story. 
There appears to be some strong, high order ARMA stuff going on. Looking at the line plot, we see that there looks to be a long seasonal component, maybe a yearly one, which would explain why the Parzen window is telling us its somewhat wandering. 
We also see what looks to be a few more peaks in the parzenwindow, which either tells us that there is a complex seasonality going on (likely at this sampling rate), or we have some sort of high order ARMA model (or potentially both). 
Lets widen our sampling rate to weeks, months, and quarters, to see if there is anything gong on there, maybe be able to catch that yearly pattern. Lets check it out:


So lets go through each of these plots and discuss:

### Weekly data


```r
wy <- sampplot(bj$w)
```

<img src="fig/unnamed-chunk-4-1.png" style="display: block; margin: auto;" />

Here we can see better that the time series is not so much wandering, as having a very long seasonal pattern (parzen window). We can tell by the little hook on the left. The ACFs are not so useful in this case, as they are tiny beyond lag 1, due to our weird sampling. However, we know for sure there is some sort of seasonal pattern going on. Potentialy there is a multiseasonal pattern again, or some high order ARMA. 

### Monthly data


```r
my <- sampplot(bj$m)
```

<img src="fig/unnamed-chunk-5-1.png" style="display: block; margin: auto;" />

Here we see another oscillating component in the ACFs, indicating again seasonal or high order ARMA (it looks a lot like patemp from tswge, which is seasonal data that is well described by a high order ARMA model). The frequency is again painting a similar picture, there is something going on with a clear frequency. lets look at the quarterly data to know for sure

### Quarterly Data


```r
sy <- sampplot(bj$s)
```

<img src="fig/unnamed-chunk-6-1.png" style="display: block; margin: auto;" />

With a bit more than 20 data points, and a supposed seasonal period of a year (4 datapoints), we would *expect* to have a frequency of about 0.2, and lo and behold, we have one. There is some strong evidence of a yearly seasonal pattern in the data (also look at the peak in the ACFs at 4, pretty good evidence right there). The scatterplot tells us a pretty obvious story as well

## Seasonal Plots

Lets check out this seasonal pattern a bit more with a seasonal plot. We will plot them all at once this time and then discuss:


```r
sda <- seasonplot(bj$day) + scale_color_hc() + theme_hc() + ggtitle("Seasonal plot: Daily")
sdw <- bj$week %>>% seasonplot + theme_hc() + scale_color_hc() + ggtitle("Seasonal plot: Weekly")
sdm <- bj$month %>>% seasonplot + theme_hc() + scale_color_hc() + ggtitle("Seasonal plot: Monthly")
sds <- bj$seas %>>% seasonplot + theme_hc() + scale_color_hc() + ggtitle("Seasonal plot: Quarterly")
plot_grid(sda, sdw, sdm, sds, ncol = 2)
```

<img src="fig/unnamed-chunk-7-1.png" style="display: block; margin: auto;" />

### Interpretation

The shape of these seasonal plots, and how they all kind of line up, indicate to me that there is some sort of seasonality. It is especially apparent in the Monthly plot, where we have it line up very well, but it is clear in the other plots as well.

> Why is this happening? Is there a real world explanation to this?

Indeed there is. Every year, around November, citywide central heating turns on inBeijing for the winter. This would cause a clear change in the air pollution, as heating is not an energy efficient process. In the summer, it goes down, because central heating is off, and people are more likely to go outside and walk and open their windows.

# Univariate analysis: setup


## S3 Generics:

To help us out a bit, lets define a few S3 generics, as well as some methods for tswge:


### Scores generic

The scores generic is used to get the score of a model of any type. 


```r
scores <- function(obj) {
    UseMethod("scores")
}
```



We will use 3 methods to evaluate  models:

### ASE


```r
ASE <- function(predicted, actual) {
    mean((actual - predicted)^2)
}
```

### MAPE


```r
MAPE <- function(predicted, actual) {
    100 * mean(abs((actual - predicted)/actual))
}
```

### Confidence Score

Confidence score is a made up metric I created, in order to evaluate the prediction interval. How it works is as follows: For each point in the actual observations, if it lies outside the prediction interval, give that point a score of 1. If it is within the confidence interval, give it a score of 0. Then, average the scores and multiply by 100 to get the percentage we got wrong.


```r
checkConfint <- function(upper, lower, actual) {
    (actual < lower) | (actual > upper)
}

confScore <- function(upper, lower, actual) {
    rawScores <- ifelse(checkConfint(upper, lower, actual), 1, 0)
    return(sum(rawScores)/(length(actual)) * 100)
}
```

### Autolot Methods

We will also add to gglots autoplot methods, so that we can view our predictions up close:



```r
.testPredPlot <- function(xs) {
    p <- ggplot() + theme_hc() + scale_color_hc()
    doplot <- function(df) {
        p <<- p + geom_line(data = df, aes(x = t, y = ppm, color = type))
    }
    out <- lapply(xs, doplot)
    out[[2]]
}
```

## Data setup:

Lets split our data into a training and test split, for univariate analysis as well:


```r
library(forecast)
library(ggplot2)
library(ggthemes)
bj <- preprocess("data/")
```

```
#> Error in na.interpolation(v, "spline") : Input x is not numeric
#> Error in na.interpolation(v, "spline") : Input x is not numeric
#> Error in na.interpolation(v, "spline") : Input x is not numeric
#> Error in na.interpolation(v, "spline") : Input x is not numeric
#> Error in na.interpolation(v, "spline") : Input x is not numeric
```

```r
bj <- bj$BeijingPM_ %>>% as.data.frame
uvar <- bj$PM_US.Post
uvar <- uvar %>>% forecast::tsclean() %>>% abs
trainU <- window(uvar, start = 3, end = c(6, 8760 - 48))
testU <- window(uvar, start = c(6, 8760 - 48))[-1]
```


# Classical Univariate Analysis

With our newfound knowledge, we will now perform a classical analysis of this time series.


```r
library(tswgewrapped)
# > my wrapper scripts of tswge, with more convenient syntax
```

Lets also define a `wge` class, and some methods:


## The wge class


```r
as.wge <- function(x) structure(x, class = "wge")
```

### Scores method for wge objects


```r
scores.wge <- function(xs) {
    mape <- MAPE(xs$f, testU)
    ase <- ASE(xs$f, testU)
    confs <- confScore(xs$ul, xs$ll, testU)
    c(ASE = ase, MAPE = mape, Conf.Score = confs)
}
```

And finally lets define an autoplot method

### Autoplot method for wge objects



```r
autoplot.wge <- function(obj) {
    testdf <- data.frame(type = "actual", t = seq_along(testU), ppm = as.numeric(testU))
    preddf <- data.frame(type = "predicted", t = seq_along(obj$f), ppm = as.numeric(obj$f))
    confdf <- data.frame(upper = obj$ul, lower = obj$ll, t = seq_along(testU))
    dfl <- list(preddf, testdf)
    .testPredPlot(dfl) + geom_line(data = confdf, aes(x = t, y = lower, alpha = 0.2), 
        linetype = 3003) + geom_line(data = confdf, aes(x = t, y = upper, alpha = 0.2), 
        linetype = 3003) + guides(alpha = FALSE)
}
```

New we can get started with our analysis:

## Differencing

First, we must pick a seasonal period for our data. As we have hourly data, and based off of previous papers on air quality, there should be a daily, weekly, and yearly seasonality in this data. We must pick one. As each of these seasonalities, other than daily, is going to be outside the range of our ACF function, we must use trail and error. First, we can eliminate a few:

An ARIMA model, with a trend, does not seem to be even slightly appropriate. This is also true of a pure ARMA model. For proof of this, please refer to `analysis/daily/classicalUvar.R`. There was no way to get an even half acceptable model with ARMA or pure ARIMA. Again through trial and error (seen in analysis/daily/classicalUvar.R), it was found that the most useful model was produced with a weekly seasonality. We will show that here:


```r
train7 <- difference(seasonal, trainU, 24 * 7)
```

<img src="fig/unnamed-chunk-14-1.png" style="display: block; margin: auto;" />

It does not appear too much like this has done anything, but in fact this differencing will allow us to model this as a high order AR model, which becomes very useful. The time series still does not appear stationary, as it definitely exhibits heteroskedacity, and probably some long term seasonality, but we will examine solving this later on. Next, we must pick an order for the time series

### A Note on the `difference` function

To prove that no smoke and mirrors were used, we will show the source code for the difference function here


```r
tswgewrapped::difference
```

```
#> function (type, x, n) 
#> {
#>     szn_trans <- function(x, n) {
#>         artrans.wge(x, phi.tr = c(rep(0, n - 1), 1))
#>     }
#>     arima_trans <- function(x, n) {
#>         f <- artrans.wge(x, phi.tr = 1)
#>         if (n == 1) {
#>             res <- f
#>             return(res)
#>         }
#>         else {
#>             arima_trans(f, n - 1)
#>         }
#>     }
#>     if (is.character(enexpr(type)) == F) {
#>         type <- as.character(enexpr(type))
#>     }
#>     if (type %in% c("arima", "ARIMA", "Arima")) {
#>         return((arima_trans(x, n)))
#>     }
#>     if (type %in% c("ARUMA", "Aruma", "aruma", "Seasonal", "seasonal")) {
#>         szn_trans(x, n)
#>     }
#> }
#> <bytecode: 0x5e97428>
#> <environment: namespace:tswgewrapped>
```



## Estimating the Order

The aic5.wge function was employed to choose the order of the model next. Aicc was chosen because it gives us the most diverse mixture of models, simpler than aic chosen models, and more complex than typically high MA order BIC models.


```r
aic72 <- aic5.wge(train7, p = 0:8, q = 0:5, type = "aicc")
```

Lets check it out


```r
aic72
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":[""],"name":["_rn_"],"type":[""],"align":["left"]},{"label":["   p"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["   q"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["       aicc"],"name":[3],"type":["dbl"],"align":["right"]}],"data":[{"1":"1","2":"4","3":"7.816267","_rn_":"11"},{"1":"5","2":"1","3":"7.816274","_rn_":"32"},{"1":"1","2":"5","3":"7.816286","_rn_":"12"},{"1":"3","2":"1","3":"7.816305","_rn_":"20"},{"1":"6","2":"0","3":"7.816310","_rn_":"37"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

Because the ACF of our data did not have any oscillations or anyting that exciting going on, just strong damping, we can go ahead and eliminate the high MA order models from this list. Because our data looks to be a high order AR model, we will go ahead and proceed with the AR(6) model (all these aiccs are basically equal).

## Estimation of parameters


```r
est7 <- estimate(train7, 6, 0)
```

```
#> 
#> Coefficients of Original polynomial:  
#> 1.1139 -0.1659 0.0165 -0.0136 0.0199 -0.0085 
#> 
#> Factor                 Roots                Abs Recip    System Freq 
#> 1-0.9560B              1.0460               0.9560       0.0000
#> 1-0.3675B+0.1548B^2    1.1872+-2.2473i      0.3934       0.1727
#> 1+0.5829B+0.1548B^2   -1.8834+-1.7073i      0.3934       0.3828
#> 1-0.3733B              2.6791               0.3733       0.0000
#>   
#> 
```

```r
phis <- est7$phi
data.frame(t(data.frame(phis, row.names = sapply(1:6, function(x) paste0("phi", 
    x)))), row.names = NULL)
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["phi1"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["phi2"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["phi3"],"name":[3],"type":["dbl"],"align":["right"]},{"label":["phi4"],"name":[4],"type":["dbl"],"align":["right"]},{"label":["phi5"],"name":[5],"type":["dbl"],"align":["right"]},{"label":["phi6"],"name":[6],"type":["dbl"],"align":["right"]}],"data":[{"1":"1.113926","2":"-0.1658656","3":"0.01647402","4":"-0.01362379","5":"0.01994103","6":"-0.008548799"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

Lets check out if we can get a time series similar to ours with that phi:


```r
recreate <- generate(arma, length(trainU), phi = phis, plot = F, sn = 34541)
par(mfrow = c(1, 2))
plot(recreate, type = "l")
plot(train7, type = "l")
```

<img src="fig/unnamed-chunk-20-1.png" style="display: block; margin: auto;" />

This looks pretty reasonable for an estimation of our parameters. Lets proceed with the tests:


```r
acf(est7$res)
```

<img src="fig/unnamed-chunk-21-1.png" style="display: block; margin: auto;" />

Nothing too outrageous in the ACF, especially for real data. However, we are clearly not telling the whole story of our data with this model, there is something left. Lets check out the results of the ljung_box test:


```r
ljung_box(est7$res, 6, 0)
```

```
#>            [,1]             [,2]            
#> test       "Ljung-Box test" "Ljung-Box test"
#> K          24               48              
#> chi.square 35.99452         103.2838        
#> df         18               42              
#> pval       0.007067426      0.0000004477291
```

It is not quite brilliant, but again this data is much more complex. We just hope that this "bad" model is somewhat useful in describing whats going on here.

### A note on the functions used

Again, no smoke and mirrors here. These are just thin wrappers for tswge, for convenience (I got a bit tired of typing `.wge` at some point).


```r
generate
```

```
#> function (type, ...) 
#> {
#>     phrase <- paste0("gen.", enexpr(type), ".wge")
#>     func <- parse_expr(phrase)
#>     eval(call2(func, ...))
#> }
#> <bytecode: 0x736c830>
#> <environment: namespace:tswgewrapped>
```

```r
estimate
```

```
#> function (xs, p, q = 0, type = "mle", ...) 
#> {
#>     if (q > 0) {
#>         return(est.arma.wge(xs, p, q, ...))
#>     }
#>     else {
#>         return(est.ar.wge(xs, p, type, ...))
#>     }
#> }
#> <bytecode: 0xe2e9020>
#> <environment: namespace:tswgewrapped>
```

```r
ljung_box
```

```
#> function (x, p, q, k_val = c(24, 48)) 
#> {
#>     ljung <- function(k) {
#>         hush(ljung.wge(x = x, p = p, q = q, K = k))
#>     }
#>     sapply(k_val, ljung)
#> }
#> <bytecode: 0x62ec4e8>
#> <environment: namespace:tswgewrapped>
```

Let us proceed with forecasting:

## Forecasting

### The `fcst` function

The `fcst` function is another wrapper for tswge's fore.*.wge:


```r
fcst
```

```
#> function (type, ...) 
#> {
#>     phrase <- paste0("fore.", enexpr(type), ".wge")
#>     func <- parse_expr(phrase)
#>     eval(expr((!!func)(...)))
#> }
#> <bytecode: 0x5966fe8>
#> <environment: namespace:tswgewrapped>
```

Lets get on with forecasting now:


```r
seafor <- fcst(aruma, s = 24 * 7, phi = est7$phi, theta = 0, n.ahead = 72, x = trainU)
```

<img src="fig/unnamed-chunk-25-1.png" style="display: block; margin: auto;" />

We have too much data to see how well this forecast actually did, so we must assess it with a larger degree of granularity:

## Model Assessment

First, lets take a closer look at the forecast, to see how it compares to our 72 hour test set:


```r
seaF <- as.wge(seafor)
autoplot(seaF)
```

<img src="fig/unnamed-chunk-26-1.png" style="display: block; margin: auto;" />

Wow. This is a stunningly good forecast. I imagine it will be quite tough to beat. Lets check out the scores as well:


```r
as.data.frame(t(scores(seaF)))
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["ASE"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["MAPE"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["Conf.Score"],"name":[3],"type":["dbl"],"align":["right"]}],"data":[{"1":"18167.88","2":"353.7521","3":"18.05556"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

```r
rmseClass <- unname(sqrt(scores(seaF)[1]))
```

Again the numbers and the graphics speak for themselves. This is an excellent forecast. It does worry me a bit that the confidence interval missed a lot, and conceptually, we may be missing more complex patterns in our data. However, it appears that an AR model with a 7 day seasonality does a great job at capturing the variant parts of our dataset, while it may not be so good for the smoother parts.

We will next begin with a multiseasonal analysis of the time series, in order to gain an even deeper insight into the dataset. 


# Multiseasonal Analysis

In our previous, classical, uniseasonal analysis, we modeled the weekly pattern in the data as a seasonality, and then attemted to model the rest with a high orde AR model. Here, we will find other ways of dealing with the complexity of hourly data. Hourly data generally (and agreeing with our EDA) has a daily, weekly, and yearly trend. We now will look for ways of modeling this complex seasonality, to make a (hopefully) better model. First, we must recast our data as `msts` objects:


```r
toMsts <- function(x) {
    msts(x, seasonal.periods = c(24, 24 * 7, 8760))
}

trainU <- toMsts(trainU)
testU <- toMsts(testU)
```

## S3 methods 

As we will be using the `forecast` library, lets set up a casting method, an autoplot method, and a scoring method for forecasts from this library:


```r
as.fore <- function(x) structure(x, class = "fore")

autoplot.fore <- function(obj) {
    testdf <- data.frame(type = "actual", t = seq_along(testU), ppm = (testU))
    preddf <- data.frame(type = "predicted", t = seq_along(testU), ppm = (obj$mean[1:length(testU)]))
    
    confdf <- data.frame(upper = obj$upper[1:length(testU), 2], lower = obj$lower[1:length(testU), 
        2], t = seq_along(testU))
    dfl <- list(preddf, testdf)
    .testPredPlot(dfl) + geom_line(data = confdf, aes(x = t, y = lower, alpha = 0.2), 
        linetype = 3003) + geom_line(data = confdf, aes(x = t, y = upper, alpha = 0.2), 
        linetype = 3003) + guides(alpha = FALSE)
}

scores.fore <- function(obj) {
    mape <- MAPE(as.numeric(obj$mean[1:length(testU)]), as.numeric(testU))
    ase <- ASE(as.numeric(obj$mean[1:length(testU)]), as.numeric(testU))
    confs <- confScore(as.numeric(obj$upper[1:length(testU), 2]), as.numeric(obj$lower[1:length(testU), 
        2]), as.numeric(testU))
    c(ASE = ase, MAPE = mape, Conf.Score = confs)
}
```

# Multiseasonal Dynamic Harmonic Regression

The first method we will use is ***dynamic harmonic regression***. Sounds very technical and scary, but what does this actually mean?

ARIMA cannot in its own right handle multiple seasonalities (and tends to struggle with long seasons), so we must find a way around. What if we could find a way to represent the seasonal patterns of the dataset, and use them as X regressors in our model? This may allow us to get a more useful model.

## A look under the hood


In class, we learned that you can perform standard regression on a time series, then apply our arima analysis on the errors of that time series, and then adjust for that. The `auto.Arima` function does all of this for you, if you supply it with an x regressor. The math behid it is basically representing the equation as follows

$$y_t = \beta_0 + \beta_1 x_{1,t} + ... + \beta_k x_{k,t} + \eta_t$$

where $\eta_t$ is a time series of errors. By estimating that time series, we can then properly estimate $y_t$, as well as make forecasts.

In the case of multiple seasonalities, we can get a bit more creative. We can represent the time series as a fourier series, that is we can write a bunch of sine and cosine terms, a set for our weekly seasonality, and a set for our yearly seasonality. We can observe this by fourier expanding our test set, adding the proper terms together, and then viewing them in comparison to the test set:


```r
exampleFourier <- fourier(testU, K = c(10, 20, 100))
library(dplyr)
shortEx <- exampleFourier %>% data.frame %>% dplyr::select(ends_with("24"))
midEx <- exampleFourier %>% data.frame %>% dplyr::select(ends_with("168"))
longEx <- exampleFourier %>% data.frame %>% dplyr::select(ends_with("8760"))
pls <- data.frame(rowSums(shortEx), rowSums(midEx), rowSums(longEx), testU)
par(mfrow = c(2, 2))
lapply(pls, plot, type = "l")
```

<img src="fig/unnamed-chunk-30-1.png" style="display: block; margin: auto;" />

```
#> $rowSums.shortEx.
#> NULL
#> 
#> $rowSums.midEx.
#> NULL
#> 
#> $rowSums.longEx.
#> NULL
#> 
#> $testU
#> NULL
```

We see here how the shape of the test set can be represented by combining each of these smaller parts. This is what we will be using as our X regressors here.




## Building the Model

First, we must go ahead and fourier expand our training set. In general, you want to set `K` to be a bit less than half the seasonal period of your time series. However, with this quantity of data, and the need to generate forecasts in a reasonable amount of time, we will only use a small expansion of each time series. This should still provide a very reasonable forecast:


```r
trainExpand <- fourier(trainU, K = c(10, 20, 100))
```

Next, we go ahead and fit the model. Note the other parameters we put in. First, we set `lambda` to 0. This lambda is the lambda for the `box-cox` transformation. For more on the box-cox transformation, please refer to [this link](https://otexts.com/fpp2/transformations.html). At a value of zero, it is a logarithmic transformation applied to our time series. What this will do for us is prevent our forecasts from going below zero (which would not make much sense). Next, because we are using fourier terms for our seasonality, we set `seasonal = FALSE`. Finally, we are ready to model (note this will take a few hours, on my laptop with 12 Cores it took 3).




```r
mseaDay <- auto.arima(trainU, xreg = trainExpand, seasonal = FALSE, lambda = 0)
```

Lets check out the unit roots of our model, as well as a summary:


```r
autoplot(mseaDay)
```

<img src="fig/unnamed-chunk-35-1.png" style="display: block; margin: auto;" />

Looks like we have an AR(5) model with our seasonal expansion. This is a slightly simpler model than our previous ARIMA(6,0,0) with a seasonality of 168hours. 

## Forecasting

We will forecast off of our training set, and compare to the testing set. Note that in this case, we will just forecast using the proper number of rows of our xregressor. In this case, it will be the last 72 values. `forecast` with an x regressor always returns something with the same length as the the length of the x regressor. This returns the same results as forecasting wiith the entire dataset, or even more surprisingly, with the test dataset.


```r
mseasFor <- forecast(mseaDay, xreg = trainExpand[(nrow(trainExpand) - 71):(nrow(trainExpand)), 
    ])
```

Lets check it out:


```r
autoplot(mseasFor)
```

<img src="fig/unnamed-chunk-37-1.png" style="display: block; margin: auto;" />

Whoa, we cant even see it. Lets plot it in the long run to see what sort of series our little trick can produce:


```r
forecast(mseaDay, xreg = trainExpand[1:(8760/2), ]) %>% autoplot
```

<img src="fig/unnamed-chunk-38-1.png" style="display: block; margin: auto;" />

That looks super realistic, amazing. Lets now zoom in on our forecast to take a deeper look:


```r
autoplot(as.fore(mseasFor))
```

<img src="fig/unnamed-chunk-39-1.png" style="display: block; margin: auto;" />

## Model Assessment

It appears that we did not capture as much of the variance, but we captured a lot more the smooth parts of the time series, especialy that lower trough. Lets put this to numbers:


```r
data.frame(t(scores(as.fore(mseasFor))))
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["ASE"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["MAPE"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["Conf.Score"],"name":[3],"type":["dbl"],"align":["right"]}],"data":[{"1":"18419.81","2":"117.2476","3":"27.77778"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

```r
rmseMsea <- unname(sqrt(scores(as.fore(mseasFor))[1]))
```




# TBATS 

TBATS is a new (to me) and exciting method for dealing with time series with complex seasonalities. TBATS stands for: ***Trigonometric seasonality, Box-Cox transformation, ARMA errors, Trend and Seasonal components***

> What?

Basically, the TBATS algorithm is going to:

* Perform a box-cox transformation on the time series

* Break it into Seasonal, ARMA (noise/residual), and Trend (with damping) components

* Estimate the Seasonal component(s) as a fourier series

* Estimate the trend component

* Estimate the ARMA component

* Combine everything back together to represent the series

> How is this different from what we just did

Unlike what we just did with dynamic harmonic regression, where the seasonality was fixed (as it is simply read in from the attributes of the `msts` object), seasonality here is allowed to change over time, as it is calculated dynamically from the data. This means that that pattern we saw in our EDA in the quarterly season plots can be maybe accounted for (where year one and two had a different pattern than the rest of the years). 

> Why havent i heard of this

Unlike a lot of time series techniques, tbats is from the 21st century, and not well proven. It also has a hard time sometimes, as it is an automated modeling tool, so some special cases will mess it up. It also is exceptionally slow, without using parallel processing it took about 20 minutes to run, on this small amount of data. Despite this, one of its great strengths is its automation, making it very easy to get a pretty good model, a lot of the time.

Lets go ahead and get into it.

## Model Building

First, lets note whether train and test are msts or ts objects, just for sanity


```r
sapply(list(trainU, testU), class)
```

```
#>      [,1]   [,2]  
#> [1,] "msts" "msts"
#> [2,] "ts"   "ts"
```

Ok great we are good to go.


```r
cores <- parallel::detectCores()
# > [1] 12
bjbats <- tbats(trainU, use.parallel = TRUE, num.cores = cores - 1)
```



### Viewing our model


```r
autoplot(bjbats)
```

<img src="fig/unnamed-chunk-44-1.png" style="display: block; margin: auto;" />


This is interesting too, so we see the I think trend that the algorithm smoothed out of the time series, as well as the seasonal fourier series it made. Lets check out that forecast:

## Forecasting


```r
batF <- forecast(bjbats, h = 72)
autoplot(batF)
```

<img src="fig/unnamed-chunk-45-1.png" style="display: block; margin: auto;" />

Looks like we cant see our forecast at all. Lets try zooming in on this with our autoplot method:
 


```r
autoplot(as.fore(batF))
```

<img src="fig/unnamed-chunk-46-1.png" style="display: block; margin: auto;" />

This is an interesting forecast. Looks like it kind of smoothed out the trend of the time series as a whole. Lets look to assess how it did:

### Model Assessment

We can use our scores method to assess the model:


```r
scores(as.fore(batF)) %>% t %>% data.frame
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["ASE"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["MAPE"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["Conf.Score"],"name":[3],"type":["dbl"],"align":["right"]}],"data":[{"1":"22434.52","2":"255.8904","3":"18.05556"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

```r
rmseBat <- unname(sqrt(scores(as.fore(batF))[1]))
```

This also seems to be on par with the past few models we've made. However, I think this model does the best job of getting the long term shape.

# Discussion and Comparison of Univariate Models

Lets now discuss all of our univariate models, and compare their strengths and weaknesses. First, lets view all of them on the same plot:


```r
uniPlot <- autoplot(ts(testU)) + labs(y = "PPM") + autolayer(ts(batF$mean), 
    series = "TBATS") + autolayer(ts(seafor$f), series = "ARUMA") + autolayer(ts(mseasFor$mean), 
    series = "Harmonic Regression") + theme_hc() + scale_color_hc(palette = "darkunica")
uniPlot
```

<img src="fig/unnamed-chunk-48-1.png" style="display: block; margin: auto;" />

Next, lets set ourselves up to view them individually:


```r
uniVars <- list(test.data = testU, TBATS = batF$mean, ARUMA = seafor$f, harmonic = mseasFor$mean)
uniMelt <- uniVars %>% lapply(function(x) ts(x, frequency = 1)) %>% data.frame %>% 
    mutate(time = 1:72) %>% tidyr::gather_(key = "series", value = "PPM", names(.[-5]))
```

Now we can start playing around.

## ARUMA 


```r
library(ggfocus)
uniMeltP <- ggplot(uniMelt, aes(x = time, y = PPM, color = series)) + geom_line()
ggfocus(p = uniMeltP, var = series, focus_levels = c("test.data", "ARUMA"), 
    focus_aes = c("color", "alpha"), alpha_focus = 1, alpha_other = 0.15) + 
    theme_hc()
```

<img src="fig/unnamed-chunk-50-1.png" style="display: block; margin: auto;" />

First looking at the ARUMA series, we find that it does an excellent job meeting the peaks of our series. However, when our series is stable, ARUMA seems to struggle. This is liely because it is relating to sequences from the past (the nature of the seasonal time forecast). However, it tends to do a pretty good job. 

## Harmonic


```r
ggfocus(p = uniMeltP, var = series, focus_levels = c("test.data", "harmonic"), 
    focus_aes = c("color", "alpha"), alpha_focus = 1, alpha_other = 0.15) + 
    theme_hc()
```

<img src="fig/unnamed-chunk-51-1.png" style="display: block; margin: auto;" />

We see this forecast, in contrast to the previous, still retains some of the shape (see the little bump at time = 20), but overall scales down the wiggles of the time series. This is because of the nature of our fourier expansion. It does seem to line up incredibly well with the overall trend of the dataset, especially towards the end, again due to the shape of the fourier representation of the series.

## TBATS


```r
ggfocus(p = uniMeltP, var = series, focus_levels = c("test.data", "TBATS"), 
    focus_aes = c("color", "alpha"), alpha_focus = 1, alpha_other = 0.15) + 
    theme_hc()
```

<img src="fig/unnamed-chunk-52-1.png" style="display: block; margin: auto;" />

TBATS, due to its roots and similarity to exponential smoothing, does not catch the details of the series very well. However, it estimates the overall seasonal trend incredibly well. 

## Overal discussion

All of these models seem to have some degree of utility, however none of them paint a complete picture. Below is the scores of each model:


```r
batCast <- as.fore(batF)
arumaCast <- as.wge(seafor)
harmonCast <- as.fore(mseasFor)
seriesL <- list(tbats = batCast, aruma = arumaCast, harmonic = harmonCast)
vapply(seriesL, scores, FUN.VALUE = double(3)) %>% t %>% data.frame
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":[""],"name":["_rn_"],"type":[""],"align":["left"]},{"label":["ASE"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["MAPE"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["Conf.Score"],"name":[3],"type":["dbl"],"align":["right"]}],"data":[{"1":"22434.52","2":"255.8904","3":"18.05556","_rn_":"tbats"},{"1":"18167.88","2":"353.7521","3":"18.05556","_rn_":"aruma"},{"1":"18419.81","2":"117.2476","3":"27.77778","_rn_":"harmonic"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

We can see the results of our performance very nicely in the table above. We see that the TBATS forecast performed worse than the the ARUMA and harmonic forecasts, but the prediction interval of the harmonic forecast was the least useful. The harmonic forecast was off by the least in general (MAPE), but did not capture the big movements (ASE), so it missed big a few times. In contrast, the ARUMA forecast was in general less close to the truth (MAPE), but captured the extreme cases much more nicely(ASE). 

To get closer to the truth, let us now examine the time series with other variables, to see if they can help us make an even better forecast:

# Multivariate Setup

First, let us grab our data frame of variables for beijing, and remove the not useful variables:


```r
names(bj)
```

```
#>  [1] "No"              "year"            "month"          
#>  [4] "day"             "hour"            "season"         
#>  [7] "PM_Dongsi"       "PM_Dongsihuan"   "PM_Nongzhanguan"
#> [10] "PM_US.Post"      "DEWP"            "HUMI"           
#> [13] "PRES"            "TEMP"            "cbwd"           
#> [16] "Iws"             "precipitation"   "Iprec"
```

```r
bj <- bj %>% dplyr::select(-c(No, starts_with("PM_D"), starts_with("PM_N")))
str(bj)
```

```
#> 'data.frame':	52584 obs. of  14 variables:
#>  $ year         : int  2010 2010 2010 2010 2010 2010 2010 2010 2010 2010 ...
#>  $ month        : int  1 1 1 1 1 1 1 1 1 1 ...
#>  $ day          : int  1 1 1 1 1 1 1 1 1 1 ...
#>  $ hour         : int  0 1 2 3 4 5 6 7 8 9 ...
#>  $ season       : int  4 4 4 4 4 4 4 4 4 4 ...
#>  $ PM_US.Post   : Time-Series  from 1 to 7: 129 148 159 181 138 ...
#>  $ DEWP         : Time-Series  from 1 to 7: -21 -21 -21 -21 -20 -19 -19 -19 -19 -20 ...
#>  $ HUMI         : Time-Series  from 1 to 7: 43 47 43 55 51 47 44 44 44 37 ...
#>  $ PRES         : Time-Series  from 1 to 7: 1021 1020 1019 1019 1018 ...
#>  $ TEMP         : Time-Series  from 1 to 7: -11 -12 -11 -14 -12 -10 -9 -9 -9 -8 ...
#>  $ cbwd         : Factor w/ 4 levels "cv","NE","NW",..: 3 3 3 3 3 3 3 3 3 3 ...
#>  $ Iws          : Time-Series  from 1 to 7: 1.79 4.92 6.71 9.84 12.97 ...
#>  $ precipitation: Time-Series  from 1 to 7: 0 0 0 0 0 0 0 0 0 0 ...
#>  $ Iprec        : Time-Series  from 1 to 7: 0 0 0 0 0 0 0 0 0 0 ...
```

Next, lets write a function to quickly split up the time series. Note the use of the `<<-` superassignment operator. This means we can reset the state of the series whenever we want.


```r
splitMvar <- function(start = 3, end = c(6, 8760 - 48)) {
    startInd <- length(window(bj$PM_US, end = start))
    endInd <- length(window(bj$PM_US, end = end))
    bjnots <- purrr::discard(bj, is.ts)
    bjnots <- data.frame(lapply(bjnots, as.factor))
    bjts <- purrr::keep(bj, is.ts)
    bjts$PM_US.Post <- uvar  # just because it is already cleaned
    notstrain <- bjnots[startInd:endInd, ]
    tsTrain <- data.frame(lapply(bjts, function(x) window(x, start = start, 
        end = end)))
    trainM <<- cbind(tsTrain, notstrain)
    notstest <- bjnots[(endInd + 1):nrow(bjnots), ]
    tsTest <- data.frame(lapply(bjts, function(x) window(x, start = c(6, 8760 - 
        47))))
    testM <<- cbind(tsTest, notstest)
}
splitMvar()
trainM
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
  </script>
</div>

```r
testM
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":[""],"name":["_rn_"],"type":[""],"align":["left"]},{"label":["PM_US.Post"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["DEWP"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["HUMI"],"name":[3],"type":["dbl"],"align":["right"]},{"label":["PRES"],"name":[4],"type":["dbl"],"align":["right"]},{"label":["TEMP"],"name":[5],"type":["dbl"],"align":["right"]},{"label":["Iws"],"name":[6],"type":["dbl"],"align":["right"]},{"label":["precipitation"],"name":[7],"type":["dbl"],"align":["right"]},{"label":["Iprec"],"name":[8],"type":["dbl"],"align":["right"]},{"label":["year"],"name":[9],"type":["fctr"],"align":["left"]},{"label":["month"],"name":[10],"type":["fctr"],"align":["left"]},{"label":["day"],"name":[11],"type":["fctr"],"align":["left"]},{"label":["hour"],"name":[12],"type":["fctr"],"align":["left"]},{"label":["season"],"name":[13],"type":["fctr"],"align":["left"]},{"label":["cbwd"],"name":[14],"type":["fctr"],"align":["left"]}],"data":[{"1":"294.0000","2":"-8","3":"79","4":"1032","5":"-5","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"0","13":"4","14":"NE","_rn_":"52513"},{"1":"301.0000","2":"-8","3":"85","4":"1032","5":"-6","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"1","13":"4","14":"SE","_rn_":"52514"},{"1":"315.0000","2":"-7","3":"85","4":"1032","5":"-5","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"2","13":"4","14":"cv","_rn_":"52515"},{"1":"326.0000","2":"-6","3":"85","4":"1032","5":"-4","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"3","13":"4","14":"NW","_rn_":"52516"},{"1":"316.0000","2":"-7","3":"85","4":"1031","5":"-5","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"4","13":"4","14":"NE","_rn_":"52517"},{"1":"337.0000","2":"-8","3":"92","4":"1030","5":"-7","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"5","13":"4","14":"NW","_rn_":"52518"},{"1":"301.0000","2":"-8","3":"92","4":"1030","5":"-7","6":"1.78","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"6","13":"4","14":"NW","_rn_":"52519"},{"1":"215.0000","2":"-8","3":"92","4":"1030","5":"-7","6":"0.45","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"7","13":"4","14":"cv","_rn_":"52520"},{"1":"201.0000","2":"-7","3":"92","4":"1030","5":"-6","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"8","13":"4","14":"NW","_rn_":"52521"},{"1":"179.0000","2":"-7","3":"85","4":"1030","5":"-5","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"9","13":"4","14":"NE","_rn_":"52522"},{"1":"165.0000","2":"-5","3":"79","4":"1030","5":"-2","6":"2.68","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"10","13":"4","14":"NE","_rn_":"52523"},{"1":"196.0000","2":"-7","3":"59","4":"1029","5":"0","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"11","13":"4","14":"cv","_rn_":"52524"},{"1":"236.0000","2":"-8","3":"50","4":"1028","5":"1","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"12","13":"4","14":"NW","_rn_":"52525"},{"1":"245.0000","2":"-7","3":"51","4":"1027","5":"2","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"13","13":"4","14":"cv","_rn_":"52526"},{"1":"264.0000","2":"-6","3":"51","4":"1026","5":"3","6":"1.34","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"14","13":"4","14":"cv","_rn_":"52527"},{"1":"318.0000","2":"-6","3":"51","4":"1026","5":"3","6":"2.23","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"15","13":"4","14":"cv","_rn_":"52528"},{"1":"360.0000","2":"-6","3":"51","4":"1026","5":"3","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"16","13":"4","14":"SE","_rn_":"52529"},{"1":"407.0000","2":"-6","3":"68","4":"1026","5":"-1","6":"1.78","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"17","13":"4","14":"SE","_rn_":"52530"},{"1":"447.0000","2":"-5","3":"74","4":"1027","5":"-1","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"18","13":"4","14":"cv","_rn_":"52531"},{"1":"494.1835","2":"-6","3":"79","4":"1027","5":"-3","6":"1.78","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"19","13":"4","14":"cv","_rn_":"52532"},{"1":"516.0732","2":"-6","3":"79","4":"1028","5":"-3","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"20","13":"4","14":"SE","_rn_":"52533"},{"1":"499.0000","2":"-6","3":"85","4":"1028","5":"-4","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"21","13":"4","14":"cv","_rn_":"52534"},{"1":"472.0000","2":"-4","3":"86","4":"1028","5":"-2","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"22","13":"4","14":"SE","_rn_":"52535"},{"1":"470.0000","2":"-7","3":"92","4":"1028","5":"-6","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"23","13":"4","14":"cv","_rn_":"52536"},{"1":"462.1651","2":"-7","3":"92","4":"1028","5":"-6","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"0","13":"4","14":"NE","_rn_":"52537"},{"1":"418.0000","2":"-6","3":"92","4":"1028","5":"-5","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"1","13":"4","14":"NW","_rn_":"52538"},{"1":"460.0000","2":"-7","3":"92","4":"1028","5":"-6","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"2","13":"4","14":"NE","_rn_":"52539"},{"1":"331.0000","2":"-7","3":"92","4":"1028","5":"-6","6":"3.58","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"3","13":"4","14":"NE","_rn_":"52540"},{"1":"228.0000","2":"-6","3":"92","4":"1028","5":"-5","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"4","13":"4","14":"NW","_rn_":"52541"},{"1":"173.0000","2":"-6","3":"85","4":"1028","5":"-4","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"5","13":"4","14":"cv","_rn_":"52542"},{"1":"45.0000","2":"-6","3":"92","4":"1029","5":"-5","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"6","13":"4","14":"NW","_rn_":"52543"},{"1":"13.0000","2":"-5","3":"74","4":"1029","5":"-1","6":"5.81","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"7","13":"4","14":"NW","_rn_":"52544"},{"1":"10.0000","2":"-6","3":"85","4":"1030","5":"-4","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"8","13":"4","14":"NE","_rn_":"52545"},{"1":"8.0000","2":"-6","3":"63","4":"1031","5":"0","6":"3.13","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"9","13":"4","14":"NW","_rn_":"52546"},{"1":"12.0000","2":"-7","3":"47","4":"1031","5":"3","6":"7.15","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"10","13":"4","14":"NW","_rn_":"52547"},{"1":"13.0000","2":"-11","3":"30","4":"1031","5":"5","6":"16.09","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"11","13":"4","14":"NW","_rn_":"52548"},{"1":"9.0000","2":"-11","3":"28","4":"1031","5":"6","6":"23.24","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"12","13":"4","14":"NW","_rn_":"52549"},{"1":"14.0000","2":"-11","3":"26","4":"1030","5":"7","6":"31.29","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"13","13":"4","14":"NW","_rn_":"52550"},{"1":"14.0000","2":"-11","3":"28","4":"1030","5":"6","6":"38.44","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"14","13":"4","14":"NW","_rn_":"52551"},{"1":"11.0000","2":"-11","3":"28","4":"1030","5":"6","6":"46.49","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"15","13":"4","14":"NW","_rn_":"52552"},{"1":"8.0000","2":"-11","3":"30","4":"1030","5":"5","6":"53.64","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"16","13":"4","14":"NW","_rn_":"52553"},{"1":"6.0000","2":"-11","3":"32","4":"1031","5":"4","6":"57.66","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"17","13":"4","14":"NW","_rn_":"52554"},{"1":"15.0000","2":"-11","3":"34","4":"1031","5":"3","6":"61.68","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"18","13":"4","14":"NW","_rn_":"52555"},{"1":"17.0000","2":"-11","3":"46","4":"1032","5":"-1","6":"63.47","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"19","13":"4","14":"NW","_rn_":"52556"},{"1":"20.0000","2":"-10","3":"54","4":"1033","5":"-2","6":"66.60","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"20","13":"4","14":"NW","_rn_":"52557"},{"1":"22.0000","2":"-10","3":"50","4":"1034","5":"-1","6":"70.62","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"21","13":"4","14":"NW","_rn_":"52558"},{"1":"33.0000","2":"-11","3":"58","4":"1034","5":"-4","6":"73.75","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"22","13":"4","14":"NW","_rn_":"52559"},{"1":"26.0000","2":"-11","3":"53","4":"1034","5":"-3","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"23","13":"4","14":"NE","_rn_":"52560"},{"1":"28.0000","2":"-11","3":"62","4":"1034","5":"-5","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"0","13":"4","14":"NW","_rn_":"52561"},{"1":"27.0000","2":"-9","3":"73","4":"1034","5":"-5","6":"3.58","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"1","13":"4","14":"NW","_rn_":"52562"},{"1":"24.0000","2":"-11","3":"73","4":"1034","5":"-7","6":"5.37","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"2","13":"4","14":"NW","_rn_":"52563"},{"1":"23.0000","2":"-11","3":"67","4":"1034","5":"-6","6":"8.50","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"3","13":"4","14":"NW","_rn_":"52564"},{"1":"19.0000","2":"-11","3":"73","4":"1034","5":"-7","6":"10.29","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"4","13":"4","14":"NW","_rn_":"52565"},{"1":"14.0000","2":"-11","3":"73","4":"1034","5":"-7","6":"12.08","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"5","13":"4","14":"NW","_rn_":"52566"},{"1":"19.0000","2":"-12","3":"72","4":"1034","5":"-8","6":"15.21","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"6","13":"4","14":"NW","_rn_":"52567"},{"1":"25.0000","2":"-11","3":"73","4":"1034","5":"-7","6":"18.34","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"7","13":"4","14":"NW","_rn_":"52568"},{"1":"22.0000","2":"-11","3":"67","4":"1034","5":"-6","6":"20.13","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"8","13":"4","14":"NW","_rn_":"52569"},{"1":"25.0000","2":"-8","3":"68","4":"1035","5":"-3","6":"23.26","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"9","13":"4","14":"NW","_rn_":"52570"},{"1":"29.0000","2":"-9","3":"50","4":"1035","5":"0","6":"26.39","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"10","13":"4","14":"NW","_rn_":"52571"},{"1":"31.0000","2":"-10","3":"43","4":"1035","5":"1","6":"28.18","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"11","13":"4","14":"NW","_rn_":"52572"},{"1":"40.0000","2":"-10","3":"37","4":"1033","5":"3","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"12","13":"4","14":"cv","_rn_":"52573"},{"1":"43.0000","2":"-11","3":"34","4":"1032","5":"3","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"13","13":"4","14":"NW","_rn_":"52574"},{"1":"48.0000","2":"-10","3":"35","4":"1031","5":"4","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"14","13":"4","14":"SE","_rn_":"52575"},{"1":"58.0000","2":"-11","3":"32","4":"1031","5":"4","6":"3.58","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"15","13":"4","14":"SE","_rn_":"52576"},{"1":"69.0000","2":"-10","3":"37","4":"1031","5":"3","6":"4.47","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"16","13":"4","14":"SE","_rn_":"52577"},{"1":"91.0000","2":"-10","3":"43","4":"1030","5":"1","6":"5.36","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"17","13":"4","14":"SE","_rn_":"52578"},{"1":"114.0000","2":"-10","3":"58","4":"1030","5":"-3","6":"6.25","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"18","13":"4","14":"SE","_rn_":"52579"},{"1":"133.0000","2":"-8","3":"68","4":"1031","5":"-3","6":"7.14","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"19","13":"4","14":"SE","_rn_":"52580"},{"1":"169.0000","2":"-8","3":"63","4":"1030","5":"-2","6":"8.03","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"20","13":"4","14":"SE","_rn_":"52581"},{"1":"203.0000","2":"-10","3":"73","4":"1030","5":"-6","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"21","13":"4","14":"NE","_rn_":"52582"},{"1":"212.0000","2":"-10","3":"73","4":"1030","5":"-6","6":"1.78","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"22","13":"4","14":"NE","_rn_":"52583"},{"1":"235.0000","2":"-9","3":"79","4":"1029","5":"-6","6":"2.67","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"23","13":"4","14":"NE","_rn_":"52584"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

Lets double check that it worked:


```r
sum(abs(as.numeric(testM$PM_US) - as.numeric(testU)))
```

```
#> [1] 0
```

```r
nrow(testM) - length(testU)
```

```
#> [1] 0
```

```r
sum(abs(trainM$PM_US - trainU))
```

```
#> [1] 0
```

```r
nrow(trainM) - length(trainU)
```

```
#> [1] 0
```

Excellent, now we can proceed with our EDA

# Multivariate EDA

Before we get into the analysis, we must do EDA, so we can understand this new information that has been given to us. We will first just plot all the time series in the dataset, to see if we notice any interesting patterns:


```r
library(tidyverse)
plotAllTs <- function(df) {
    tsdf <- keep(df, is.ts)
    lapply(tsdf, function(x) autoplot(x) + theme_hc())
}
plot_grid(plotlist = plotAllTs(trainM), labels = names(keep(trainM, is.ts)))
```

<img src="fig/unnamed-chunk-57-1.png" style="display: block; margin: auto;" />

We already learned some interesting stuff. First of all, precipitation does not seem to be in any way similar to our dataset. So, we probably wont use it as a predictor, as it also does not make logical sense. Of all the time seres here, humidity appears to have a similar trend pattern to our target. Pressure looks similar but a bit off, so at a high lag maybe it is interesting. Dewpoint has the wrong period, and is a function of temperature and humidity, so most likely it does not provide us with any interesting information.

## Analysis of Categorical Variables:

Lets see how each of the categorical variables affect our air content:


```r
catTable <- function(cat) {
    trainM %>% arrange(!!sym(cat)) %>% group_by(!!sym(cat)) %>% summarise(meanPPM = mean(PM_US.Post))
}

map(names(discard(trainM, is.ts)), catTable) %>% pander
```



  *

    ----------------
     year   meanPPM
    ------ ---------
     2012    89.91

     2013    100.6

     2014    97.36

     2015    81.27
    ----------------

  *

    -----------------
     month   meanPPM
    ------- ---------
       1      130.4

       2      118.9

       3      104.2

       4      81.41

       5      76.97

       6      80.25

       7      72.96

       8      63.12

       9      66.98

      10      103.3

      11      101.8

      12      108.8
    -----------------

  *

    ---------------
     day   meanPPM
    ----- ---------
      1     87.95

      2     76.97

      3     72.52

      4     70.35

      5     81.04

      6     96.23

      7     95.08

      8     95.85

      9     92.02

     10     77.49

     11     76.2

     12     73.52

     13     94.75

     14     88.16

     15     104.5

     16     99.5

     17     96.24

     18     93.28

     19      103

     20     99.4

     21     102.9

     22     99.53

     23     106.7

     24     96.15

     25     101.9

     26     104.2

     27     96.54

     28     106.6

     29     94.39

     30     86.17

     31     91.18
    ---------------

  *

    ----------------
     hour   meanPPM
    ------ ---------
      0      103.6

      1      103.1

      2      101.4

      3      98.01

      4      94.18

      5      91.81

      6      90.26

      7      90.15

      8      88.86

      9      87.51

      10     86.73

      11     84.65

      12     83.38

      13     81.49

      14     80.67

      15     80.91

      16     82.57

      17     86.23

      18     91.22

      19     96.89

      20     100.9

      21     102.8

      22     103.8

      23     104.1
    ----------------

  *

    ------------------
     season   meanPPM
    -------- ---------
       1       87.6

       2       72.02

       3       90.83

       4       119.5
    ------------------

  *

    ----------------
     cbwd   meanPPM
    ------ ---------
      cv     118.4

      NE     82.67

      NW     62.05

      SE     104.6

      NA     75.6
    ----------------


<!-- end of list -->

There are a few variables which stand out, namely hour, season, and cbwd (wind direction). Hour appears to be a daily and nigthly pattern, where at night, the average PPM goes up. Lets create a new categorical variable for daytime nighttime:


```r
library(forcats)
hoursToDayNight <- function(df) {
    df[["hour"]] %>% fct_collapse(night = c(as.character(18:23), as.character(0:5)), 
        day = as.character(6:17))
}

trainM$dayNight <- hoursToDayNight(trainM)
testM$dayNight <- hoursToDayNight(testM)
catTable("dayNight")
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["dayNight"],"name":[1],"type":["fctr"],"align":["left"]},{"label":["meanPPM"],"name":[2],"type":["dbl"],"align":["right"]}],"data":[{"1":"night","2":"99.31249"},{"1":"day","2":"85.28434"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

An astounding and noticeable change between day and night. This will be an absolutely useful factor. As for wind direction, we have a lot of NAs, and we only have 4 wind directions, so that may not be so useful. Season will definitely be useful.

## Another look at our numeric variables:

Lets look at the relationship between our numeric variables and air quality now, using scatterplots:

  
  ```r
  plotVsResponse <- function(x) {
      plot(trainM$PM_US.Post ~ trainM[[x]], xlab = x)
      lw1 <- loess(trainM$PM_US.Post ~ trainM[[x]])
      j <- order(trainM[[x]])
      lines(trainM[[x]][j], lw1$fitted[j], col = "red", lwd = 3)
  }
  trainM2 <- trainM %>% select(-contains("prec"))  # precipitation is all 0
  trainM2 %>% keep(is.numeric) %>% names %>% walk(plotVsResponse)
  ```
  
  <img src="fig/unnamed-chunk-60-1.png" style="display: block; margin: auto;" /><img src="fig/unnamed-chunk-60-2.png" style="display: block; margin: auto;" /><img src="fig/unnamed-chunk-60-3.png" style="display: block; margin: auto;" /><img src="fig/unnamed-chunk-60-4.png" style="display: block; margin: auto;" /><img src="fig/unnamed-chunk-60-5.png" style="display: block; margin: auto;" /><img src="fig/unnamed-chunk-60-6.png" style="display: block; margin: auto;" />

All of these variables appear to have some sort of relationship with the air quality, so we will have to investigate further. 

## CCF analysis

Lets look at the cross correlation between all the useful variables next:


```r
ppm <- trainM$PM_US.Post
ccfPlot <- function(x) {
    ccf(ppm, trainM[[x]], main = x)
}
trainM2 %>% keep(is.numeric) %>% names %>% walk(ccfPlot)
```

<img src="fig/unnamed-chunk-61-1.png" style="display: block; margin: auto;" /><img src="fig/unnamed-chunk-61-2.png" style="display: block; margin: auto;" /><img src="fig/unnamed-chunk-61-3.png" style="display: block; margin: auto;" /><img src="fig/unnamed-chunk-61-4.png" style="display: block; margin: auto;" /><img src="fig/unnamed-chunk-61-5.png" style="display: block; margin: auto;" /><img src="fig/unnamed-chunk-61-6.png" style="display: block; margin: auto;" />

It appears we have a very complex correlation structure. It is important to note that the cross correlation of dewpint does look like some sort of combination of humidity and temperature, which makes physical sense. Now that we have completed our EDA, lets go ahead and get to doing analysis.

# Multivariate Models

## Data Preparation

Lets go ahead and add to our `splitMvar` function, and automatically set ourselves up for the daily and nightly encoded categorical variable. We are encoding it as a number from 0 to 1, so it will play nicely with *all* our models, including the LSTM later on.


```r
hoursToDayNight <- function(df) {
    df[["hour"]] %>% fct_collapse(night = c(as.character(18:23), as.character(0:5)), 
        day = as.character(6:17)) %>% as.numeric %>% -1
}
```

Next lets modify our splitting function:


```r
splitMvar <- function(start = 3, end = c(6, 8760 - 48)) {
    startInd <- length(window(bj$PM_US, end = start))
    endInd <- length(window(bj$PM_US, end = end))
    bjnots <- purrr::discard(bj, is.ts)
    bjnots <- data.frame(lapply(bjnots, as.factor))
    bjnots$dayNight <- hoursToDayNight(bjnots)
    bjts <- purrr::keep(bj, is.ts)
    bjts$PM_US.Post <- uvar  # just because it is already cleaned
    notstrain <- bjnots[startInd:endInd, ]
    tsTrain <- data.frame(lapply(bjts, function(x) window(x, start = start, 
        end = end)))
    trainM <<- cbind(tsTrain, notstrain)
    notstest <- bjnots[(endInd + 1):nrow(bjnots), ]
    tsTest <- data.frame(lapply(bjts, function(x) window(x, start = c(6, 8760 - 
        47))))
    testM <<- cbind(tsTest, notstest)
    trainM[c(7:14)] <<- NULL
    testM[c(7:14)] <<- NULL
}
splitMvar()
head(trainM)
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":[""],"name":["_rn_"],"type":[""],"align":["left"]},{"label":["PM_US.Post"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["DEWP"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["HUMI"],"name":[3],"type":["dbl"],"align":["right"]},{"label":["PRES"],"name":[4],"type":["dbl"],"align":["right"]},{"label":["TEMP"],"name":[5],"type":["dbl"],"align":["right"]},{"label":["Iws"],"name":[6],"type":["dbl"],"align":["right"]},{"label":["dayNight"],"name":[7],"type":["dbl"],"align":["right"]}],"data":[{"1":"303","2":"-12","3":"72","4":"1030","5":"-8","6":"0.89","7":"0","_rn_":"17521"},{"1":"215","2":"-13","3":"78","4":"1031","5":"-10","6":"1.79","7":"0","_rn_":"17522"},{"1":"222","2":"-13","3":"72","4":"1032","5":"-9","6":"3.58","7":"0","_rn_":"17523"},{"1":"85","2":"-13","3":"72","4":"1033","5":"-9","6":"6.71","7":"0","_rn_":"17524"},{"1":"38","2":"-13","3":"49","4":"1033","5":"-4","6":"4.92","7":"0","_rn_":"17525"},{"1":"23","2":"-14","3":"45","4":"1034","5":"-4","6":"10.73","7":"0","_rn_":"17526"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>


# VAR

Our first multivariate technique is going to be VAR. One of the interesting properties of VAR is what it means about the data. VAR is telling us that our time series are *endogenous*, that is to say that they all affect each other. Now with our weather patterns, this makes sense. Weather is a complex, dynamic system of complex feedback cycles, so it is natural that all the weather variables are endogenous to each other. However; let us discuss how our air particulate content relates to the weather, to determine weather a VAR model is physically appropriate

## Discussion of Appropriateness

In this section we will discuss how each of our weather patterns interplay (or dont) with our air quality.

### Humidity

#### Effect of Humidity on Air Quality

Humidity has a strong effect on air quality. On a humid day, water in the air will "stick" to air particulates. This weighs them down and makes them much larger, causing them to stick around for longer. This will cause the air particulates to hang around in one place. 

It is also important to note that this has an effect on temperature. When water sticks to air pollution, energy from the sun is reflected off of the water, imparting a bit of energy to the water and spreading the sunlight some more. This adds to the haze appearance when there is a lot of pollution. Over time, due to water's high specific heat, should cause the air to heat up, as the small amount of extra energy stored in the water attached to the pollution will be transferred quickly to the air. (Note this should also have an affect on pressure slowly, because $P \propto T$).

#### Effect of Air Quality on Humidity

Surprisingly, air quality also has a positive feedback with humidity. This is due to the "sticking" effect we discussed earlier. Heavy air particulate matter with water attached causes other air particles to stick around too. You can actually prove this with an experiment.


Supplies

* An aerosol, such as hairspray

* A jar with a lid

* Ice

* Warm water

Take the warm water and put it in your jar. Put the lid on upside down with ice in it, for about 30 seconds (this causes the evaporated water in the air to condense a bit, raising the humidity). 

Now, lift the ice lid up, and spray a tiny bit of your aerosol in the jar and quickly replave the ice lid. 

Then, watch a cloud form inside the jar.

You can do this experiment with any air particulate, for example smoke from a match also works nicely.

It is safe to say that a bidirectional VAR forecast of humidity is appropriate.



### Temperature

#### How Temperature Relates to Surface Air Quality

Temperature has a clear effect on air quality. Not only does cold air cause air particulates to stick around more (slower moving), but also more condensed water particles in the air, which we already know about.

Similarly, in the winter, tropospheric temperature inversions can occur when the stratosphere (think, up high) gets heated more than the troposphere (we live here). Normally, the air naturally convects because it is hotter in the troposphere than the stratosphere, but in the winter, with our long nights, the opposite can occur. This is a temerature inversion. This causes the convection to stop, and the air to remain trapped there. This traps the pollutants in the same place, and causes them to accumilate

#### Effect of air quality on temperature

The air quality + humidity should have a slight effect on temperature, but global weather patterns are more powerful

#### Discussion

It is appropriate to say that temperature affects air quality, but only slightly appropriate to say the opposite

### Dewpoint

Dewppoint is simply a function of humidity and temperature, so this variable will not be included in our model, as it is simply overfitting.

### Wind speed

This is a complex relationship, but basically it does have some sort of complex relationship with air pollution. When it is windy, pollutants can be moved thousands of miles, while when it is still, they can accumulate. So it depends, but it does have an effect. Heavy particulte matter may also slow the wind down, but it is unclear and unproven. Overall though, it is somewhat appropriate to do this bidirectional forecast.

### Pressure

Given the complex relationship between pressure, temperature, humidity, and wind speed, which are all somewhat related to air quality in interesting ways, it is safe to include pressure in our VAR models as well

### Precipitation

No matter how much i think, I cannot say that you can predict surface air quality with precipitation or vice versa, so this is likely inappropriate

## Setup

### Data preporation

Lets go ahead and separate our columns into endogenous and exogenous variables:


```r
vartrain <- trainM %>% dplyr::select(-c(dayNight))
varexo <- matrix(trainM$dayNight, dimnames = list(NULL, "dayNight"))
```

We also need to define a function to make new exogenous predictors:


```r
makeExo <- function(n) {
    day <- c(rep(0, 5), rep(1, 12), rep(0, 7))
    return(matrix(rep(day, n), dimnames = list(NULL, "dayNight")))
}
```


### S3 methods

We need three S3 methods here, an `as` method, an `autoplot` method, and a `scores` method:


```r
as.var <- function(x) structure(x, class = "var")
autoplot.var <- function(obj) {
    us <- obj$fcst$PM_US.Post
    testdf <- data.frame(type = "actual", t = seq_along(testM$PM_US.Post), ppm = as.numeric(testM$PM_US.Post))
    preddf <- data.frame(type = "predicted", t = seq_along(testM$PM_US.Post), 
        ppm = (obj$fcst$PM_US.Post[, 1]))
    dfl <- list(testdf, preddf)
    confdf <- data.frame(t = seq_along(testM$PM_US.Post), upper = us[, 3], lower = us[, 
        2])
    .testPredPlot(dfl) + geom_line(data = confdf, aes(x = t, y = lower, alpha = 0.2), 
        linetype = 3003) + geom_line(data = confdf, aes(x = t, y = upper, alpha = 0.2), 
        linetype = 3003) + guides(alpha = FALSE)
}
scores.var <- function(obj) {
    mape <- MAPE(obj$fcst$PM[, 1], testM$PM_US)
    ase <- ASE(obj$fcst$PM[, 1], testM$PM_US)
    us <- obj$fcst$PM_US.Post
    conf <- confScore(upper = us[, 3], lower = us[, 2], testM$PM_US)
    c(ASE = ase, MAPE = mape, Conf.Score = conf)
}
```

Now, we can go ahead and pick variables

## Linear Modeling

An important part in choosing variables for VAR is to model our data in the same fassion as a linear model (for reference on how to choose variables for VAR, please refer to [this link](https://cran.r-project.org/web/packages/vars/vignettes/vars.pdf)).


```r
summary(lm(trainM))
```

```
#> 
#> Call:
#> lm(formula = trainM)
#> 
#> Residuals:
#>     Min      1Q  Median      3Q     Max 
#> -175.56  -50.10  -13.17   32.25  378.96 
#> 
#> Coefficients:
#>                Estimate  Std. Error t value             Pr(>|t|)    
#> (Intercept) 1312.562192   79.364221  16.538 < 0.0000000000000002 ***
#> DEWP           0.573997    0.171460   3.348             0.000816 ***
#> HUMI           0.971564    0.054544  17.812 < 0.0000000000000002 ***
#> PRES          -1.212078    0.077217 -15.697 < 0.0000000000000002 ***
#> TEMP          -3.171640    0.169294 -18.734 < 0.0000000000000002 ***
#> Iws           -0.281490    0.009344 -30.125 < 0.0000000000000002 ***
#> dayNight       8.395574    0.868097   9.671 < 0.0000000000000002 ***
#> ---
#> Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
#> 
#> Residual standard error: 76.63 on 34985 degrees of freedom
#> Multiple R-squared:  0.2303,	Adjusted R-squared:  0.2302 
#> F-statistic:  1745 on 6 and 34985 DF,  p-value: < 0.00000000000000022
```