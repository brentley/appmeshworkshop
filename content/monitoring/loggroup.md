---
title: "Review CloudWatch log groups"
date: 2018-09-18T16:01:14-05:00
weight: 15
---

* Access CloudWatch Logs to check the existence of the **appmesh-workshop-frontrend-envoy** and **appmesh-workshop-crystal-envoy** log groups.

![log group](/images/monitoring/log_group.png)

* Take a few minutes to review the logging details produced by the Envoy container.

![log group](/images/monitoring/log_stream.png)

Let's query the log group using CloudWatch Insights.

* Select CloudWatch Insights.

![log group](/images/monitoring/insights_1.png)

* Select the **appmesh-workshop-crystal-envoy** log group.

![log group](/images/monitoring/insights_2.png)

* Replace the exiting query with the following one.

```
parse @message '[*] "* * *" * * * * * * "*" "*" "*" "*" "*"' as startTime, method, envoyOriginalPath, protocol, responseCode, responseFlags, bytesReceived, bytesSent, duration, upstreamServiceTime, xForwardedFor, userAgent, requestId, host, upstreamHost
| sort startTime desc
| filter envoyOriginalPath like /^(?!\s*$).+/
| stats count(responseCode) by envoyOriginalPath, responseCode
```

* This query will parse the Envoy access logs, and count the number of response codes per request path. Run the query to get your results.

![log group](/images/monitoring/insights_3.png)