---
author: "Stefano Boriero" 
title: "SLO practice with SpringBoot and Prometheus"
date: "2024-09-01"
description: "A practical guide to build SLOs for Java microservices using SpringBoot and Prometheus"
tags: [ "observability"]
categories: ["slo"]
---

This article explains Service Level Objectives (SLOs), a monitoring practice aimed at ensuring systems provide an acceptable level of service. As with any practice, it won't make systems reliable on its own; developers need to follow it consistently for it to be fruitful. It requires discipline and time to become familiar with it, but it's a rewarding journey that you can add to your CV. If you're passionate about the topic, feel free to read the complete [Google SRE books](https://sre.google/books/); if not, we'll link specific chapters in the following paragraphs that will give you a good grasp of the subject.

## Key concepts
The core concepts of this practice are SLI, SLO, and SLA. An excellent introduction to these concepts is the [the "Service Level Objective" chapter from Google SRE book](https://sre.google/sre-book/service-level-objectives/). Below we will provide our own definitions along with practical examples.

### SLI - Service Level Indicator
A Service Level Indicator is **the quantitative measure being monitored**. For availability and latency-focused SLI’s, this is expressed in terms of a ratio of good events over total events over a given period of time. A “good” event is one that can be classified as offering an acceptable level of service. This is a very powerful abstraction presented in [the "Implementing SLOs" chapter](https://sre.google/workbook/implementing-slos/).

$$
SLI = \frac{good\_events_{over\_period\_of\_time}}{total\_events_{over\_period\_of\_time}}
$$

### SLO - Service Level Objective
A Service Level Objective is **the target set for the SLI that is aimed to be achieved**. For availability and latency-focused SLI’s, which are expressed as a ratio, the __objective__ itself is a percentage.

$$
SLO = SLI > target_{SLO}
$$

### SLA - Service Level Agreement
A Service Level Agreement is a legal obligation to meet a given target for the SLI. Usually it's worded like the SLO, but with a more relaxed target. This is to ensure that even if we breach our SLO, we still have headroom to address the issue before it has legal repercussions on the business.

# Defining a SLO
The most important question to ask when defining an SLO is: *What does a good event look like for my system?* This is often not purely a technical question and usually requires collaboration with the Product team. Sometimes the answer can be simple. For example, with an availability SLO, a good event might be any response other than a server error. For an SLO based on response latency, you’ll need to define the boundary between a good user experience and a bad one. Should it be the same for all endpoints, or should different endpoints have their own definitions of good service? There is no absolute answer; it depends on the specific service.

The other parameter you need to choose is the period over which to evaluate whether you’ve met your target. This is typically 28 or 30 days. Longer periods help make the framework more resilient to transient failures.

## A practical example: latency based SLO
To better understand these concepts, let's define an SLO for a web service’s latency. A __good event__ could be defined as a request successfully served in under 200ms, and the __total events__ would be the number of requests successfully served.

> [**NOTE**] For latency is useful to consider only successful requests (with status code 2xx) since 3xx and 4xx, while still being valid responses, they usually have much lower latency since they execute little business logic in our services. Filtering them out leads to less skewed data.

Next, we pick the evaluation window, let's say 30 days. With these, we have enough to define our SLI

$$
SLI = \frac{number\_of\_succesful\_requests\_served\_under\_200ms_{30\_days}}{number\_of\_successful\_requests\_served_{30\_days}}
$$

Lastly, you need to pick the target objective that you want your SLI to meet: in our example we'll use 90\%.

$$
SLO = SLI > 0.9 = \frac{number\_of\_succesful\_requests\_served\_under\_200ms_{30\_days}}{number\_of\_successful\_requests\_served_{30\_days}} > 0.9
$$

If we want to express this in natural language, we could describe our SLO like the following:

> 90\% of succesful requests to the service are served in under 200ms, over a 30 day period.

> [**NOTE**] Mathematically, an SLO formulated in the above structure is equivalent to say that we want the 90th percentile of latency to be less than 200ms.


Something to note here is that the SLO target is `90%`, not `200ms`. `200ms` is what defines _a good level of service_ and it is an important parameter to set in synergy with Product: with this SLO definition, we aim to provide a good level of service `90%` of the time.

### Reporting the necessary data with SpringBoot and Micrometer
Now that we have defined what our SLO looks like, we have to instrument our system to report the necessary data. In our example specifically, we have to report the number of requests that are served under 200ms. The next paragraph is going to focus on how to do this with SpringBoot and Micrometer, since it's the technical framework for the majority of OSP applications. SpringBoot actuator has powerful features that make it trivial to report this information, you just need to:

1. Import the [Spring Actuator](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#actuator) dependency.
2. Configure [SLO distribution](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html#actuator.metrics.customizing.per-meter-properties) for the `http.server.requests` metric. This will instruct Micrometer to create a histogram where each SLO boundary will become a bucket. With the following configuration, Micrometer will report a histogram with two buckets (the `+Inf` bucket is always created automatically as an upper bound) with the respective boundaries. **You must adjust the thresholds according to your SLO definition**, since Micrometer will collect information only around the thresholds defined.

```yaml
  management.metrics.distribution:
    slo.http.server.requests: 200ms
```

Once you add these depenency and the configuration, deploy your app and it should start exposing the necessary data to perform SLOs calculations.

#### Explore the data with Prometheus and Grafana
The metric `http.server.requests` is a [standard observation metric](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#actuator.metrics.supported.spring-mvc) in Spring Boot apps that collects information about requests served by your application. By adding the SLO distribution configuration, this metric will have an additional label `le` (short for less than or equal) that we can use to get the number of requests that took less than the specified time to be served. You can supply only the values that you defined in your SLO distribution configuration, that's why it's important to specify them judiciously.

Building on top of the example above, this query will plot the number of requests served under 200ms over a period of 28 days.

```code
sum(
    increase(
        http_server_requests_seconds_bucket{
            service_name="myservice",
            outcome="SUCCESS",
            le="0.2"}[30d]
        )
    )
```

Connecting with our example, this is is calculating the _number of successful requests served in under 200ms, over a 30 days window_. If we want to know the total number of successful requests served, we can just change the `le` selector to `le="+Inf"`: this will compute the number of successful requests that were served with any latency. Or in other words _total number of successful requests served over a 30 days window_.

```code
sum(
    increase(
        http_server_requests_seconds_bucket{
            service_name="myservice",
            outcome="SUCCESS",
            le="+Inf"}[30d]
        )
    )
```

Given this data, we can compute our SLI over 30 days just dividing the two:

```code
sum(
    rate(
        http_server_requests_seconds_bucket{
            service_name="myservice",
            outcome="SUCCESS",
            le="0.2"}[30d]
        )
    )
    \
sum(
    rate(
        http_server_requests_seconds_bucket{
            service_name="myservice",
            outcome="SUCCESS",
            le="+Inf"}[30d]
        )
    )
```
> [**WARNING**] Notice how in the last expression we used `rate` instead of `increase`? That's because `increase` is just syntactic sugar for a `rate` multiplied by the rate period. When dividing the 2 sums, the multiplication carried out by the `increase` function can be mathematically symplified, so the result will be equivalent and more precise by using `rate` directly.

> [**NOTE**] Aggregation over large windows of time require the data to be present for the whole aggregation period. This means that until you have 28 days of data available, the result of the following queries might look off. For a quicker feedback if you're headed in the right direction, you can replace 28d with a smaller time window, like 1h.

# Alerting on SLO
The rationale of the SLO framework is to ensure our systems provide a minum level of service. To meet those SLOs it is vital to have some alerting in place that fires when they are at risk, for us to take action and correct the failure. This is a distillation of the key takeaways from the [Google's SRE workbook alerting chapter](https://sre.google/workbook/alerting-on-slos/) on the desirable properties of alerts. When defining your alerting strategy, try to keep them in mind.

- Have alerting on both fast and slow burning rates.
    - Fast burn rate alerts are going to be useful to detect serious service outages that have a noticeable impact on your provided level of service. Slow burn rate alerts are going to detect degradations of smaller magnitude that protract for longer period of time.
- Set realistic targets.
    - Analyse your current service behaviour and level provided, and set a target that you can confidently reach. You don't want your alerting to be too noisy right off the bat, or you'll get used to ignore it.
- Balance detection and reset times:
    - Get notified early is important so you can act on the incident quickly. However short recall time of the alerts after you deploy a fix is also important and avoids getting used to see alerts firing which may drive to overlook it.
- Try to keep things consistent as much as possible.
    - Given defining good alerts requires quite a bit of configuration, having different alerts configured with different parameters might become a burden. Try to categorise and group them into clusters of alerts with similar properties and reuse the same alerting parameters for all SLO in the cluster.

## Error Budget
We don't have to meet our SLO for each and every request. Error budget represents how much we can afford to not meet the SLO: each time a request does not meet our SLO, it consumes our error budget. That is why it is often reported the percentage of error budget still avaiable: this means that if the error budget is 0%, another event failing to meet our requirement will make us fail to meet our SLO, bringing the error budget below 0%. On the other hand, an error budget of 100\% means that every single event over the slo window qualified as a _good event_. The formula to calculate the error budget left is:

$$
error\_budget = \frac{SLI_{slo\_window} - target_{SLO}}{1 - target_{SLO}}
$$

Where $SLI_{slo\_window}$ is the SLI evaluated over the full SLO time window, usually 30 days.

> [**NOTE**] Error budgets expressed as percentages can have negative values! When that happens, it means that we did not meet our SLO over the target time window.

## Burn rate
Burn rate measures how fast we're burning through our error budget, over a given period of time. So _burn rate_ must always be measured in the context of a chosen evaluation period, $burn\_rate_{1h}$ or $burn\_rate_{1d}$. Often we'll be intersted to know what was the burn rate over several periods of time, shorter ones like the last hour which are often referred to as _Fast burn rates_, or longer periods of time like the last couple of days which are referred to as _Slow burn rate_. The formula to compute the burn rate over a given time window is the following:

$$ burn\_rate_{time\_window} = \frac{1 - SLI_{time\_window}}{1 - target_{SLO}}$$

Where $SLI_{time\_window}$ is the SLI evaluated over the shorter time window we're interested in, like 1 hour or 1 day. 

When the burn rate over the chosen time window has value 1, it means that _if we keep burning the budget at this rate_, at the end of the slo window we'll be left with exactly 0\% error budget left. Sustained values greater than 1 will lead to not meeting the SLO target over the full slo window. Rather than trying to interpret burn rate values, it's much easier to reason about the situation you want to uncover: usually you want to know when you consumed X\% of your error budget in a given time window. For example, we want to know what's the burn rate that signals when we consumed 2

$$
burn\_rate\_threshold = \frac{slo\_window}{time\_window} \times budget\_consumption\_percentage
$$

This may sound complex, but hopefully the practical example will make things clearer with a more hands on approach.

> [**NOTE**] Burn rate is an important concept in SLO alerting, as most alerts will be written based on it. Burn rate functions as a proxy that predicts wether the behaviour over the evaluated burn rate time window could in fact result in a breach of the SLO over the total slo window.

## A practical example: latency based SLO
Let's follow on with our example

> 90\% of succesful requests to the service are served in under 200ms, over a 28 day period.

Let's try defining multi-window and multi-burn-rate alert. We want to get paged when 2% of the error budget has been consumed in one hour (the [Fast burn rate alert](#fast-burn-rate-alert)) and be warned when we consumed 10\% of the error budget in one day (the [Slow burn rate alert](#slow-burn-rate-alert)).

### Fast burn rate alert
Fast burn alert aim at detecting outages that consume our error budget quickly and require prompt intervention. Let's then define an alert that would page us when 2\% of our error budget is consumed in one hour. First, we need to evaluate the SLIs over the chosen time windows. We can use the same calculations presented [in the explore the data paragraph](#explore-the-data-with-prometheus-and-grafana) but with a different time window. Additionally, in order to make alert definition simpler and easy to read, we can leverage [recording rules](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/#recording-rules) to precompute data aggregations.

```yaml
namespace: myservice-slo-recording-rules
groups:
  - name: Fast burn rate
    rules:
      - record: myservice:latency_sli:1h
        expr: |
          sum(rate(http_server_requests_seconds_bucket{
                  service_name="myservice",
                  outcome="SUCCESS",
                  le="0.2",
                  }[1h]
          )) /
          sum(rate(http_server_requests_seconds_bucket{
                  service_name="myservice",
                  outcome="SUCCESS",
                  le="+Inf",
                  }[1h]
          ))
      # This is an extra aggregation that is going to give faster recall times for our alerts
      - record: myservice:latency_sli:5m
        expr: |
          sum(rate(http_server_requests_seconds_bucket{
                  service_name="myservice",
                  outcome="SUCCESS",
                  le="0.2",
                  }[5m]
          )) /
          sum(rate(http_server_requests_seconds_bucket{
                  service_name="myservice",
                  outcome="SUCCESS",
                  le="+Inf",
                  }[5m]
          ))
```

The next step is calculating the actual thresholds for the alert. For that, let's use the formula presented in the [key concept paragraph](#burn-rate). Remember to convert times variables to the same unit of time! In our case, 30 days is equivalent to 720 hours.

$$
burn\_rate\_threshold = \frac{slo\_window}{time\_window} \times budget\_consumption\_percentage = \frac{720}{1} \times 2\% = 720 \times \frac{2}{100} = 14.4
$$

With this, we can go ahead and define the alert as the following:
```yaml
groups:
  - name: SLO Rules
    rules:
      - alert: Fast burn rate
        annotations:
          description: SLO Fast burn rate alert.
          summary: Service is fast burning its error budget.
        expr: |
          (
            (1 - myservice:latency_sli:1h) / (1 - 0.9) > 14.4
          and
            (1 - myservice:latency_sli:5m) / (1 - 0.9) > 14.4
          )
        labels:
          severity: critical
          app_group: pharos
```

Defined this way, our fast burn alert will fire when burn rate is sustained over 13.43 for both 1 hour and 5 minutes time windows. Thanks to the former time window, alert will cease fire after 5 minutes the burn rate is below the threshold bringing a better reset time. Without it, we would need to wait a whole hour after the solution is put in place before our alert resolves.

### Slow burn rate alert
Slow burn rate alerts aim at uncovering issues that might not result in big outages but, protacting over longer period of times, will lead to total consumption of error budget. While they might not require prompt intervention (i.e. being paged in the middle of the night) it's important to be notified about them and intervene as part as your normal working hours duties. For this example, let's set up an alert that will notify us when 10\% of our error budget is consumed over 1 day. Similarly to the above example, let's evaluate the SLI over these longer time windows:

```yaml
namespace: myservice-slo-recording-rules
groups:
  - name: Slow burn rate
    rules:
      - record: myservice:latency_sli:1d
        expr: |
          sum(rate(http_server_requests_seconds_bucket{
              service_name="myservice",
              outcome="SUCCESS",
              le="+Inf",
              }[1d]))
            /
          sum(rate(http_server_requests_seconds_bucket{
              service_name="myservice",
              outcome="SUCCESS",
              le="+Inf",
              }[1d]))
```

Again, let's calculate the burn rate thresholds with the chosen parameters:
$$
burn\_rate\_threshold = \frac{slo\_window}{time\_window} \times budget\_consumption\_percentage = \frac{720}{24} \times 10\% = 30 \times \frac{10}{100} = 3
$$

The resulting alert definition is the following:
```yaml
groups:
  - name: SLO Rules
    rules:
      - alert: Slow burn rate
        annotations:
          description: SLO Slow burn rate alert.
          summary: Service is slowly burning its error budget.
        expr: |
          (
            (1 - myservice:latency_sli:1d) / (1 - 0.9) > 3
          and
            (1 - myservice:latency_sli:1h) / (1 - 0.9) > 3
          )
        labels:
          severity: warning
          app_group: pharos
```

> [**NOTE**] The recorded metric `myservice:latency_sli:1h` was already defined for the [Fast burn rate alert](#fast-burn-rate-alert), so we can reuse that here without defining it again as a recorded rule.
