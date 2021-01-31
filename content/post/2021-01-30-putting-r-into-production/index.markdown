---
title: Putting R Into Production
author: Matt Kaye
date: '2021-01-30'
slug: []
categories:
  - data-science
tags:
  - R
  - data science
lastmod: '2021-01-30T17:30:34-07:00'
keywords: []
description: ''
comment: no
toc: yes
autoCollapseToc: no
postMetaInFooter: no
hiddenFromHomePage: no
contentCopyright: no
reward: no
mathjax: no
mathjaxEnableSingleDollar: no
mathjaxEnableAutoNumber: no
hideHeaderAndFooter: no
flowchartDiagrams:
  enable: no
  options: ''
sequenceDiagrams:
  enable: no
  options: ''
---

## Introduction

As promised, this post is going to cover how we productionize R at CollegeVine. As I said before, I'm exclusively using R for my data science work at CV, which means that we needed to come up with a way to get over the "R is hard to productionize!" obstacle. Here's how we do it.

## Building a Modeling Pipeline

What I've found to be the most important piece of productionizing the work that I do is having a rock-solid pipeline set up to make everyone's life easier. At CollegeVine, this requires a few simple steps.

### An Internal R Package

The most important step is making an internal R package to speed up your workflow. I got this idea from a Baltimore Orioles co-worker, and ran with it on my first day at CV. Initially, `collegeviner`, our R package for data science at CV, contained a few functions like `connect_to_***_db()`, which would return a connection to one of a few CV databases. Since the, it's grown enormously. Now, we have functions to common data cleaning steps, setting up a connection to Slack from R with `slackr`, common queries that we run all the time, implementations of statistical tests, and, most importantly, a whole bunch of functions for productionizing models. Those productionizing functions include:

* `save_to_production(..., stage = '', bucket = '')`, which takes some R objects like data, cross validation results, a finalized model, etc. and saves it to an S3 bucket by calling `aws.s3::s3save()` on the back end. The most important piece of this function aside from not needing to directly call anything in `aws.s3` is that it enforces some strict naming conventions so that the objects being saved are always saved in a specified format. For us, that format looks like this: `final_model_timestamp_validated.Rdata`. This means that every model we have in production is stamped exactly like that, meaning there's no room for confusion around how objects are named and we can use this strict naming convention for other helpful functions, such as:
* `get_latest_***(bucket = '')`, which loads the latest data, tuning results, validated model, etc. from the specified S3 bucket. Again, this means that we're keeping our naming conventions consistent and making it super easy to load data, models, etc. from S3. 
* `list_all_***(bucket = '')` lists all of the data, models, etc. in a bucket.
* `load_from_s3(objname, bucket = '')` loads a specific object from S3, so we can have more flexibility than always loading the most recent one.

### Shiny

The next key piece that we use is `Shiny`, which is a fantastic R package for building dashboard and applications directly from R. At CV, we use Shiny in production to load a model, run some pre-determined diagnostics, and then deploy it to production when we're happy with how the model is performing. For example, if we've fit a Bayesian linear regression model to predict the house prices in Ames, Iowa, we'd have a tab in our model deployment Shiny app that has some marginal effects plots, some diagnostic scores (on aggregate or broken down by arbitrary groups) like MSE, etc. The idea here is that we call `get_latest_model()` to get a model, run a bunch of diagnostics, and then, if everything looks okay, we've just set up a button that says `Verify!` that pops up a modal saying "Are you sure you want to verify this model?" with some other info. Once you verify the model, it calls `save_to_production()`, which saves the model to production and sends a Slack message alerting us that a new model was just deployed.

### Containers

So now, another question. I've mentioned how `save_to_production()` puts a finalized, verified model into S3, but how do we actually deploy it? For that, we rely on containers. We use containers for a big chunk of our data science work at CV, but most importantly for deploying Shiny apps and APIs to production. For Shiny, we build our apps with the help of `{golem}`, which is an awesome R package that makes it easy to spin up production-ready Shiny applications. For deploying APIs, we use `Plumber`, which is another great R package that makes building APIs super easy in R. Once  we have a `{golem}` or `Plumber` set up, we containerize it with Docker and deploy it to Heroku, and which point it exists on the internet and can be accessed by anyone on the team.

## Tying It All Together

To tie it all together, here is the list of steps that my R productionization workflow consists of:

1. Cleaning data / feature engineering, and saving the cleaned data to production
2. Loading the cleaned data from production and using it to tune a model
3. Saving the results of the tuning process to production and loading them to fit a final model
4. Loading the data and the final model into our productionization Shiny app to run some diagnostics and to verify it, so that it's finally sent to production.
5. Loading the model into a `plumber.R` file that sets up an API to run predictions through the model

And that's it! As it turns out, productionizing R is not so complicated after all, and there are lots of amazing tools like `Shiny` and `Plumber` at your disposal to make it easy, which work seamlessly with S3 and Heroku to make deploying R to production a breeze.
