---
title: Why I Write R, and Why You Should Too
author: Matt Kaye
date: '2021-01-14'
slug: []
categories:
  - data science
tags:
  - R
  - data science
  - programming
  - statistics
lastmod: '2021-01-14T17:41:14-07:00'
keywords: []
description: ''
comment: no
toc: yes
autoCollapseToc: no
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

I write R all the time. All of my work is done in R (yes, that includes productionizing models), I write R for fun, I maintain an R package and am the author of another, and I wrote this post in R Markdown. But, why? If you peruse blogs on [Towards Data Science](towardsdatascience.com) or [r/datascience](reddit.com/r/datascience), you'll find person after person telling new data scientists or data scientists-in-training to learn python instead of R. The reasons are numerous: Python is easier to learn, Python is a programming language while R is just statistical software, Python is better for machine learning while R is better for statistical learning (yes, that's a real quote from [this Medium post](https://medium.com/@datadrivenscience/python-vs-r-for-data-science-and-the-winner-is-3ebb1a968197)), R is dying as more people switch to Python, and, most importantly, Python is just, way, way easier to productionize. Are any of those arguments true? Well, it depends who you ask, but certainly not if you ask an R zealot like me. I'm not going to spend time discussing these points (I'll save the production one for another post). Instead, I'll take this post to try to convince you that R is awesome and worth learning and using, and isn't just that pesky software package you needed to use to pass Stat 101.

## Things I love about R

There are a whole bunch of things that I love about R. Here are a few of them:

### Functional Programming

At its core, R is a *functional programming* language. If that is jibberish to you, taking a deep dive in FP is worth the time. Learn some Haskell or F# to see why FP is so cool. The benefit of R being *mostly* functional is that we can get around one of R's biggest weaknesses: loops in R are slow. Like, really, really slow. Fortunately, there are some awesome R packages and functions that let us replace loops with much sexier approaches to solving problems wholesale. So much so that I rarely, if ever, find myself using loops in R. Maybe once every month or two, at the absolute most. Here's an example:


```r
N <- 100
numbers <- list()
for (i in 1:N) {
  numbers[[i]] <- i
}

numbers[1:5]
```

```
## [[1]]
## [1] 1
## 
## [[2]]
## [1] 2
## 
## [[3]]
## [1] 3
## 
## [[4]]
## [1] 4
## 
## [[5]]
## [1] 5
```

There's some really, really naive R to make a list of numbers from 1 to 100. Now, let's square all the numbers in that list.


```r
numbers2 <- list()
for (i in 1:length(numbers)) {
  numbers2[[i]] <- numbers[[i]]**2
}

numbers2[1:5]
```

```
## [[1]]
## [1] 1
## 
## [[2]]
## [1] 4
## 
## [[3]]
## [1] 9
## 
## [[4]]
## [1] 16
## 
## [[5]]
## [1] 25
```
And we've done it. Now `numbers2` contains the squares of the numbers in `numbers`. Just for kicks, let's time it to see how long those two loops took. There's a cool package called `tictoc` that lets us do this easily.


```r
library(tictoc)

tic(msg = 'numbers')
N <- 100
numbers <- list()
for (i in 1:N) {
  numbers[[i]] <- i
}
toc()
```

```
## numbers: 0.005 sec elapsed
```

```r
tic(msg = 'numbers2')
numbers2 <- list()
for (i in 1:length(numbers)) {
  numbers2[[i]] <- numbers[[i]]**2
}
toc()
```

```
## numbers2: 0.005 sec elapsed
```

That's pretty slow. But before, I claimed that for loops in R are slow and that FP lets us get around that, right? Let's see how we can use vectors and functions in R to solve this problem.


```r
tic()
numbers <- 1:100
numbers2 <- numbers**2
toc()
```

```
## 0.003 sec elapsed
```

That approach is dramatically faster. What about if we want to do something more complicated, like adding a column to each dataframe in a list of dataframes based on a couple of other values? Let's try it!




```r
df_list <- data.frame(a = 1:10**4 %>% as.numeric(), b = rnorm(10**4, pi * 1:10**4, 10)) %>%
  group_by(a) %>%
  group_split()

df_list_2 <- list()

tic()
for (i in 1:length(df_list)) {
  df_list_2[[i]] <- df_list[[i]] %>%
    mutate(c = a * b)
}
toc()
```

```
## 20.183 sec elapsed
```

Not great. So, what about the FP approach?


```r
# library(furrr)
df_list <- data.frame(a = 1:10**4 %>% as.numeric(), b = rnorm(10**4, pi * 1:10**4, 10)) %>%
  group_by(a) %>%
  group_split()

plan(multisession, workers = 8)
tic()
df_list <- df_list %>%
  future_map(function(x) x %>% mutate(c = a * b))
toc()
```

```
## 9.581 sec elapsed
```

Cool, so what happened here? Two things, really. First, my code got cleaner. Writing `map()` instead of `for ...` with a bunch of junk is a great way to be able to apply a function to everything inside of a list. Second, it got faster. That's because I'm parallelizing across cores, which I can do super easily when using `map()`, since I'm not explicitly relying on an index like I am with `df_list[[i]]` in the for loop. 

In my view, this is awesome. I use FP functions like `map()`, `reduce()`, `walk()`, etc. every day, because they're so much cleaner and more intuitive than looping. You can do FP in Python too, with `map()` + `lambda` (i.e. `map(lambda x: x ** 2, my_list)`) will square all of the elements of `my_list`. It's just that the code that I use every day in R to do things like data wrangling is all FP at it's core. All of the `Tidyverse` functions (i.e. lots of stuff in `dplyr`, `stringr`, `tidyr`, etc.) let you perform common data wrangling tasks with simple calls like `select()` to pick some columns to keep or `mutate()` to add new columns. These functions are, well, functional (because they're functions), and they add new columns or select columns by name and by using operations as opposed to indexing. So now, more on data wrangling.

### Data Wrangling

So, now to one of R's biggest strengths: data wrangling. Simply put, the `Tidyverse` is in a class of it's own. It's what `pandas` was built to emulate, and makes data wrangling a breeze. This is especially important to data scientists, since, as the cliche goes, we spend `insert_large_number_here` (usually 90%) percent of our time data wrangling. What does this look like? Here's something that's easy to do in R:


```r
library(gapminder)

gapminder %>%
  group_by(year, continent) %>%
  summarize(lifeExp = mean(lifeExp), .groups = 'drop_last') %>%
  mutate(worldwideLifeExp = mean(lifeExp),
         relativeToAvg = lifeExp - worldwideLifeExp)
```

```
## # A tibble: 60 x 5
## # Groups:   year [12]
##     year continent lifeExp worldwideLifeExp relativeToAvg
##    <int> <fct>       <dbl>            <dbl>         <dbl>
##  1  1952 Africa       39.1             54.5       -15.3  
##  2  1952 Americas     53.3             54.5        -1.20 
##  3  1952 Asia         46.3             54.5        -8.16 
##  4  1952 Europe       64.4             54.5         9.93 
##  5  1952 Oceania      69.3             54.5        14.8  
##  6  1957 Africa       41.3             56.7       -15.4  
##  7  1957 Americas     56.0             56.7        -0.748
##  8  1957 Asia         49.3             56.7        -7.39 
##  9  1957 Europe       66.7             56.7         9.99 
## 10  1957 Oceania      70.3             56.7        13.6  
## # … with 50 more rows
```
T
here are a few things going on here. First, we're grouping the Gapminder data by continent and year, and calculating the mean life expectancy by group. Then, we remove the last grouping factor in one swoop by using `.groups = 'drop_last'`, and add two new columns: one with the worldwide mean life expectancy in that year, and one with the difference between the continent and the mean, within year. The fact that all of this work can be done in four lines is fantastic, since it saves tons of programming time, but it also makes reading the code simple. In `dplyr`, fist, you take the `gapminder` data, then, you group it by `year` and `continent`, then, you calculate the mean life expectancy within each group, and so on. The lines of code read like how you would explain the steps to someone.

Another bonus here is that `dplyr` and `dbplyr` allow to seamless database connection, meaning that not only can you run `dplyr` commands on local data frames, but you can also run them against tables in a database, and then collect the query result when you choose to. For example:

```r
my_gapminder_database %>%
  group_by(year, continent) %>%
  summarize(lifeExp = mean(lifeExp), .groups = 'drop_last') %>%
  mutate(worldwideLifeExp = mean(lifeExp),
         relativeToAvg = lifeExp - worldwideLifeExp)
```

This code performs the exact same function as before, but `dbplyr` translates it to a SQL query, which is then executed and the result of which is returned to you. In my experience, the one line of code required to connect to a database in R is dramatically easier than, say, connecting with a Django model where you need to make a whole `models.py` file that specifies all of the columns in the database table and their data types before you can actually grab any data. Using `dbplyr` lets me pull data and work with it on the fly with almost no downtime, which saves me time and effort, and also lets me ship code much, much more quickly.

### Data Visualization

Another winner for R: `ggplot2`. `ggplot2` is another `Tidyverse` package that's far and away the best data visualization package that you can get out of the box that's easy to use and free. D3 is also free, but the learning curve is far steeper. Tableau is another fantastic paid option.


```r
library(ggthemes)
gapminder %>%
  filter(year %in% c(1967, 2007)) %>%
  ggplot(aes(x = gdpPercap, y = lifeExp, size = pop, color = continent)) +
  geom_point() + 
  facet_wrap(~year) +
  scale_x_log10() +
  theme_fivethirtyeight() +
  labs(x = 'GDP Per Capita',
       y = 'Life Expectancy',
       color = 'Continent',
       size = 'Population') +
  theme(axis.title = element_text())
```

<img src="static/rmarkdown-libsunnamed-chunk-10-1.png" width="672" style="display: block; margin: auto;" />

And just like that, in only a few lines, I have an awesome chart that's themed, has colors, has varying point sizes, and has multiple panes for different years. `ggplot` is extremely powerful and flexible, and there are other awesome `ggplot`-adjacent packages like `ggthemes` and `patchwork` for customizing your plots even farther. 

### Tidymodels

Finally, `Tidymodels` is a suite of R packages much like the `Tidyverse` that seek to make building models a breeze. The idea is that instead of needing to rely on a bunch of different packages to do your modeling (like `ranger` for random forests, `xgboost` for tree boosting, `glm` or `brms` for linear models, `knn` for nearest neighbor algorithms, and `keras` or `nnet` for fitting neural nets), you can simply let `Tidymodels` do those function calls for you. You can specify a model like this:

```r
logistic_reg() %>%
  set_engine('glm')
```

This code sets up a logistic regression model using `glm` as the backend. You can also use `keras` or `glmnet` for logistic regression:

```r
logistic_reg() %>%
  set_engine('keras')

logistic_reg() %>%
  set_engine('glmnet')
```

And the syntax is exactly the same. From there, you can use a bunch of other `Tidymodels` functions and packages like `tune` to tune your model, `recipes` to build a "recipe", which helps you set up data cleaning and preprocessing steps, and `parsnip` to fit the model object. The `Tidymodels` umbrella simplifies modeling a ton, full stop.

### Packages

Finally, packages. In short, there are a *ton* of awesome R packages for literally anything you could imagine doing. I'm an author and maintainer of `slackr`, which lets you send messages (text, plots, regression output, rendered LaTeX, etc.) to Slack directly from R. At work, I use this for error logging and notifying myself about issues in production, keeping myself posted on how cron jobs I have set up are doing, and giving our eng team simple heads-ups when a new model is deployed. It also makes it easy to make a `ggplot` and send it straight to Slack, or send some regression output to a channel in a nicely-formatted way. There are a zillion R packages that I rely on all the time. Many of them have already been mentioned here, but are worth mentioning again. Many, many thanks to the team at RStudio for their work maintaining, authoring, and constantly adding awesome features to amazing packages like `dplyr`, `ggplot2`, `tidyr`, `lubridate`, `parsnip`, etc. etc. etc. The list goes on and on. 

### RStudio + the RStudio Team

Speaking of the RStudio team, they (and RStudio itself) are the last reason I'll discuss for why I love R. RStudio is an amazing data science IDE. It makes writing code, exploring tabular data, and making plots perfectly straightforward. There are also a zillion plugins (like being able to run D3, python, etc. directly from RStudio) that make life as a data scientist in R even easier. I could go on and on about RStudio as an IDE, but just take my word for it being awesome. 

The RStudio *team* is equally amazing, and the work that they do is fantastic. It probably starts at the top with Hadley Wickham, who, at least as far as I can see, is the mad scientist responsible for conceptualizing, cooking up, and maintaining basically the entire Tidyverse. I can't go on enough about all of the amazing work that the team at RStudio does, though. From building awesome packages like the ones I mentioned earlier, to integrating R with other resources like Hugo to make `blogdown` possible (which is how I write my blog), to doing an amazing job of responding to issues and questions, to organizing awesome conferences, they really do so much great work, and the R community wouldn't be the same without them.

## Wrapping Up

Hopefully there was at least something interesting to you in this post, whether it be a package you hadn't seen before, or maybe functional programming tidbits, or anything else. This is not an exhaustive list of the reasons I love writing R, but it's a fairly good start. As promised, I'll also write a post about productionizing R (or at least how we do it at Collegevine), so those who are R-wary because of hiccups in production can rest easy: it's not as tough as people make it out to be! Until then, hopefully this was helpful or interesting,






