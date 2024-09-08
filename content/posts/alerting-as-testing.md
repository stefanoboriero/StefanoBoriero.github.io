---
author: "Stefano Boriero"
title: "The parallelism between testing and monitoring strategies"
date: "2024-08-19"
description: ""
tags: ["observability"]
---

At the beginning of my career, I was tasked with testing a ticket developed by a very senior developer on my team. When I opened the test instructions, to my surprise, I found this:

> Just deploy the changes to the staging environment. If something is not working, we'll get alerted.

It was only a few years later when I really understood the wisdom behind this sentence, and this article will try to explain why alerting should be tought as a superset of testing.

> Good *testing* aims at ensuring that *the system is behaving as expected at rest*.

> Good *monitoring* aims at ensuring that *the system is behaving as expected at runtime*.

The parallelism doesn't end here. Testing has been long discussed in Software Engineering and there are certain properties of sound testing suites that can be extended to provide a blueprint for implementing sound monitoring strategies.

> In testing, we want to avoid flaky test. In monitoring, we want to minimise false positives alerts.

Flaky tests are tests that do not always succeed, with seemingly random and inexplicable failures. When test failures rate increases, engineers start ignoring those test as they are seen as not reliable. This is problematic because often the reason for the flakiness is an actual bug. The parallelism with alerting here is with false positive alerts. A false positive alert is an alert that fires when sometimes there isn't a real problem: this has the same effect as flaky test, alerts with a high rate of false positives are destined to be ignored in the long run, even if they might be signalling a problem!

> In testing, we want to test the behaviour of the system, not its implementation. In monitoring, we want to alert on misbehaviours, not on its causes.

One of the mantras of testing is that you shouldn't be testing implementation details of your system, but instead that the system is doing what it's behaving and keeping its invariants. At the same time, we should alert on problems not causes. The reason is that when the cause manifests, it is not ensured that it will lead to an actual problem. Take a web service as an example, which expected behaviour is to serve traffic in a timely manner, that sometimes is subject to long GC pauses: if such pauses happen when the system is serving a high volume of traffic, it's likely that the system won't serve all of it in a timely manner, and you should be alerted. However if such pauses happen when no traffic is being served, you shouldn't be alerted. SLO practice help in this by providing a framework for defining acceptable levels of service for your systems and building alerts based on them.

> In testing, we want to write testeable code. In monitoring, we want to write observable code.

Sometimes is hard to write good quality testing on a codebase that is not lending itself to it. For this reason, practices like TDD are preached forcing engineers to structure their code such that testing is easy and sound. In the same way, code should be written such that is easy to instrument and measure the behaviour of the system.

In a nutshell, a good monitoring infrastructure should be in fact a continous test suite running on your system.
