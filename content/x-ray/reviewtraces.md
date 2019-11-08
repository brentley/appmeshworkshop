---
title: "Review traces"
date: 2018-09-18T16:01:14-05:00
weight: 20
---

Now access the [X-Ray console](console.aws.amazon.com/xray/home). Once in the X-Ray console, select **Service Map** from the left hand side menu. Wait a few seconds for the service map to render.

You will see something like this:

![xray](/images/x-ray/xray_1.png)

Whenever you select a node or edge on an AWS X-Ray service map, the X-Ray console shows a latency distribution histogram. Latency is the amount of time between when a request starts and when it completes. A histogram shows a distribution of latencies. It shows duration on the x-axis, and the percentage of requests that match each duration on the y-axis.

![xray](/images/x-ray/xray_2.png)