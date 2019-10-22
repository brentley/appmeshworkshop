---
title: "Update the crystal code"
date: 2018-09-18T16:01:14-05:00
weight: 5
---

In our microservices architecture, planning to handling transient faults is a common practice. Retry of a request can help overcome short lived network blips or short term interruptions on the server services due to redeployments like Service Unavailable (HTTP Error code 503), Gateway Timeout (HTTP Error code 504).

Since our application makes recurring use of network calls to other services, we will add the ability to retry connections between services using AWS App Mesh based on HTTP error codes.

Let's see how we can set retry duration and attempts within route configurations. Currently, when a request is made, our frontend Ruby service creates a header to denote the current time and send that request to the Crystal backend service. We will update the Crystal logic to respond with a 503 if the time of the original request (passed through the header) and the current time is less than 1 second. 

Initially without retry in place this error will be thrown consistently. After we introduce retries, we will begin to see the time of the original request and the current time will eventually be greater than 1 second (depending on the retry policy configuration you want to set but the example configuration will work once the route is updated during the walkthrough). At this point the Crystal backend service will return a 200 and send back the response.

* Create the following patch in your **ecsdemo-crystal** project. 

```bash
cat <<-'EOF' > ~/environment/ecsdemo-crystal/add_server_error.patch
From 48f89f671650e1827fbac5f7bb3c8ff3a06d9c73 Mon Sep 17 00:00:00 2001
From:
Date:
Subject: [PATCH] add service unavailable error if req < 1sec

---
 src/server.cr | 11 +++++++++--
 1 file changed, 9 insertions(+), 2 deletions(-)

diff --git a/src/server.cr b/src/server.cr
index b08ae32..b6875b5 100644
--- a/src/server.cr
+++ b/src/server.cr
@@ -26,8 +26,15 @@ server = HTTP::Server.new(
     ]) do |context|
       if context.request.path == "/crystal" || context.request.path == "/crystal/"
         epoch_ms = Time.now.epoch_ms
-        context.response.content_type = "text/plain"
-        context.response.print "Crystal backend: Hello! from #{az_message} commit #{code_hash} at #{epoch_ms}"
+        req_epoch_ms = context.request.headers["epoch_ms"].to_i64
+        if (epoch_ms - req_epoch_ms) > 1000
+          context.response.content_type = "text/plain"
+          context.response.print "Crystal backend: Hello! from #{az_message} commit #{code_hash} at #{epoch_ms}"
+        else
+          context.response.status_code = 503
+          context.response.headers["Content-Type"] = "text/plain"
+          context.response.puts "The server cannot handle the request"
+        end
       elsif context.request.path == "/health"
         context.response.content_type = "text/plain"
         context.response.print "Healthy!"
-- 
2.22.0


EOF
```

* First, take a look at what changes are in the patch.

```bash
git apply --stat ecsdemo-crystal/add_server_error.patch
```

* Run **git apply** to apply the patch.

```bash
sh -c 'cd ~/environment/ecsdemo-crystal && git apply add_server_error.patch'
```

* Build the container

```bash
CRYSTAL_ECR_REPO=$(jq < cfn-output.json -r '.CrystalEcrRepo')

$(aws ecr get-login --no-include-email)

docker build -t crystal-service ecsdemo-crystal
docker tag crystal-service:latest $CRYSTAL_ECR_REPO:green
docker push $CRYSTAL_ECR_REPO:green
```