---
title: "Update the Crystal code"
date: 2018-09-18T16:01:14-05:00
weight: 5
---

The first thing we need to do is to change the Crystal backend application code in order to introduce the 503 error responses. To do so, create the following patch in your **ecsdemo-crystal** project.

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
       elsif context.request.path == "/crystal/api" || context.request.path == "/crystal/api/"
         context.response.content_type = "application/json"
         context.response.print %Q({"from":"Crystal backend", "message": "#{az_message}", "commit": "#{code_hash.chomp}"})
--
2.22.0


EOF
```

First, take a look at what changes are in the patch. Note that now the application will start sending the 503 error messages.

```bash
git apply --stat ecsdemo-crystal/add_server_error.patch
```

Run **git apply** to apply the patch.

```bash
git -C ~/environment/ecsdemo-crystal apply add_server_error.patch
```

And build the container with the newest version of the Crystal application:

```bash
CRYSTAL_ECR_REPO=$(jq < cfn-output.json -r '.CrystalEcrRepo')

$(aws ecr get-login --no-include-email)

docker build -t crystal-service ecsdemo-crystal
docker tag crystal-service:latest $CRYSTAL_ECR_REPO:error
docker push $CRYSTAL_ECR_REPO:error
```
