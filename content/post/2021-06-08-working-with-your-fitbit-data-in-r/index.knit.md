---
title: Working With Your Fitbit Data in R
author: Matt Kaye
date: '2021-06-08'
slug: []
categories:
  - data science
tags:
  - R
  - data science
lastmod: '2021-06-08T21:16:59-04:00'
keywords: []
description: ''
comment: no
toc: yes
autoCollapseToc: yes
postMetaInFooter: no
hiddenFromHomePage: no
contentCopyright: no
reward: no
mathjax: yes
mathjaxEnableSingleDollar: yes
mathjaxEnableAutoNumber: yes
hideHeaderAndFooter: no
flowchartDiagrams:
  enable: no
  options: ''
sequenceDiagrams:
  enable: no
  options: ''
---



## Introduction

`fitbitr 0.1.0` is now available on CRAN! You can install it with

```r
install.packages("fitbitr")
```

or you can get the latest dev version with

```r
## install.packages("devtools")
devtools::install_github("mrkaye97/fitbitr")
```

`fitbitr` makes it easy to pull your Fitbit data into R and use it for whatever interests you: personal projects, visualization, medical purposes, etc. 

This post shows how you might use `fitbitr` to pull and visualize some of your data.

## Sleep

First, you should either generate a new token with `generate_token()` or load a cached token with `load_cached_token()`.




```r
library(fitbitr)
library(lubridate)
library(tidyverse)

# load_cached_token(".httr-oauth")
```

And then you can start pulling your data!


```r
sleep <- sleep_summary(
  start_date = today() - months(3),
  end_date = today()
)

sleep %>% 
  head()
```

```
## # A tibble: 6 x 11
##    log_id date       start_time          end_time            duration efficiency
##     <dbl> <date>     <dttm>              <dttm>                 <int>      <int>
## 1 3.12e10 2021-03-08 2021-03-07 20:49:00 2021-03-08 07:10:30 37260000         97
## 2 3.12e10 2021-03-09 2021-03-08 22:28:30 2021-03-09 07:23:30 32100000         96
## 3 3.13e10 2021-03-10 2021-03-09 22:35:30 2021-03-10 07:19:30 31440000         98
## 4 3.13e10 2021-03-11 2021-03-10 23:48:00 2021-03-11 06:51:00 25380000         95
## 5 3.13e10 2021-03-12 2021-03-11 23:08:00 2021-03-12 06:28:00 26400000         96
## 6 3.13e10 2021-03-13 2021-03-12 23:16:00 2021-03-13 07:42:30 30360000         96
## # … with 5 more variables: minutes_to_fall_asleep <int>, minutes_asleep <int>,
## #   minutes_awake <int>, minutes_after_wakeup <int>, time_in_bed <int>
```

Once you've loaded some data, you can visualize it!


```r
library(zoo)
library(scales)
library(ggthemes)

sleep <- sleep %>%
  mutate(
   date = as_date(date),
   start_time = as_datetime(start_time),
   end_time = as_datetime(end_time),
   sh = ifelse(hour(start_time) < 8, hour(start_time) + 24, hour(start_time)), #create numeric times
   sm = minute(start_time),
   st = sh + sm/60,
   eh = hour(end_time),
   em = minute(end_time),
   et = eh + em/60,
   mst = rollmean(st, 7, fill = NA), #create moving averages
   met = rollmean(et, 7, fill = NA),
   year = year(start_time)
)

sleep %>%
    ggplot(aes(x = date))+
    geom_line(aes(y = et), color = 'coral', alpha = .3, na.rm = T)+
    geom_line(aes(y = st), color = 'dodgerblue', alpha = .3, na.rm = T)+
    geom_line(aes(y = met), color = 'coral', na.rm=T)+
    geom_line(aes(y = mst), color = 'dodgerblue', na.rm=T)+
    scale_y_continuous(breaks = seq(0, 30, 2),
                       labels = trans_format(function(x) ifelse(x > 23, x - 24, x), 
                                             format = scales::comma_format(suffix = ":00", accuracy = 1))
    )+
    labs(x = "Date",
         y = 'Time')+
    theme_fivethirtyeight()+
  scale_x_date(date_breaks = '1 month', date_labels = '%b', expand = c(0, 0))+
  facet_grid(. ~ year, space = 'free', scales = 'free_x', switch = 'x') +
  theme(panel.spacing.x = unit(0,"line"), 
        strip.placement = "outside")
```

<img src="index_files/figure-html/unnamed-chunk-5-1.png" width="672" />

This bit of code makes a nicely formatted plot of the times you went to sleep and woke up over the past three months. You can also use `fitbitr` to expand the time window with a little help from `purrr` (the Fitbit API rate limits you, so you can't request data for infinitely long windows in a single request).


```r
## this will pull the past 15 months of data

start_months <- c(3, 6, 9, 12, 15)
end_months <- c(0, 3, 6, 9, 12)

sleep <- map2_dfr(
  start_months,
  end_months,
  function(x, y) sleep_summary(
    today() - months(x), 
    today() - months(y)
  )
)
```

After pulling the data, we can use the same code again to visualize it.


```r
sleep <- sleep %>%
  mutate(
   date = as_date(date),
   start_time = as_datetime(start_time),
   end_time = as_datetime(end_time),
   sh = ifelse(hour(start_time) < 8, hour(start_time) + 24, hour(start_time)), #create numeric times
   sm = minute(start_time),
   st = sh + sm/60,
   eh = hour(end_time),
   em = minute(end_time),
   et = eh + em/60,
   mst = rollmean(st, 7, fill = NA), #create moving averages
   met = rollmean(et, 7, fill = NA),
   year = year(start_time)
) %>%
  distinct()

sleep %>%
    ggplot(aes(x = date))+
    geom_line(aes(y = et), color = 'coral', alpha = .3, na.rm = T)+
    geom_line(aes(y = st), color = 'dodgerblue', alpha = .3, na.rm = T)+
    geom_line(aes(y = met), color = 'coral', na.rm=T)+
    geom_line(aes(y = mst), color = 'dodgerblue', na.rm=T)+
    scale_y_continuous(breaks = seq(0, 30, 2),
                       labels = trans_format(function(x) ifelse(x > 23, x - 24, x), 
                                             format = scales::comma_format(suffix = ":00", accuracy = 1))
    )+
    labs(x = "Date",
         y = 'Time')+
    theme_fivethirtyeight()+
  scale_x_date(date_breaks = '1 month', date_labels = '%b', expand = c(0, 0))+
  facet_grid(. ~ year, space = 'free', scales = 'free_x', switch = 'x') +
  theme(panel.spacing.x = unit(0,"line"), 
        strip.placement = "outside")
```

<img src="index_files/figure-html/unnamed-chunk-7-1.png" width="672" />

## Heart Rate and Steps

You can also pull your heart rate data with `fitbitr`. Maybe we're curious about seeing how the number of minutes spent in the "fat burn," "cardio," and "peak" zones correlates with the number of steps taken that day. Let's find out!










