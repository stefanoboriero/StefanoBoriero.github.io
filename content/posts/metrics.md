---
author: "Stefano Boriero"
title: "What is a metric?"
date: "2024-12-01"
description: ""
tags: ["observability"]
---

If you ever worked with observability, you most likely have heard about the three pillars of observability which are historically Metrics, Traces and Logs. My personal preference is to refer to them as signals, rather than pillars, so that's the term I'm going to use throughout this article.

You could work your way using them without deep diving into formal understanding of the differences between these signals, however really understanding their nuances can prove invaluable to instrument effectively your systems. In this article we're going to focus on metrics.

## Definition of a metric

> A metric is a measurement taken at **a given point in time**, at a regular interval and often **aggregated**
