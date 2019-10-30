---
title: "Update the Crystal code"
date: 2018-09-18T16:01:14-05:00
weight: 5
---

In a canary release deployment, total traffic is separated at random into a production release and a canary release with a pre-configured ratio. Typically, the canary release receives a small percentage of the traffic and the production release takes up the rest. The updated service features are only visible to the traffic through the canary. You can adjust the canary traffic percentage to optimize test coverage or performance.

Remember, in App Mesh, every version of a service is ultimately backed by actual running code somewhere (Fargate tasks in the case of Crystal), so each service will have it's own virtual node representation in the mesh that provides this conduit.

Additionaly, there is the physical deployment of the application itself to a compute environment. Both Crystal deployments will run on ECS using the Fargate launch type. Our goal is to test with a portion of traffic going to the new version, ultimately increasing to 100% of traffic.

* Create the following patch in your **ecsdemo-crystal** project.

```bash
cat <<-'EOF' > ~/environment/ecsdemo-crystal/add_time_ms.patch
From beb964253b921a0a34ed45bac0ab052667523441 Mon Sep 17 00:00:00 2001
From:
Date:
Subject: [PATCH 1/2] add canary hash

---
 code_hash.txt | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)

diff --git a/code_hash.txt b/code_hash.txt
index a7c041f..dbd1857 100644
--- a/code_hash.txt
+++ b/code_hash.txt
@@ -1 +1 @@
-NOHASH
+CANARY
--
2.22.0


From 23a8640575f0fcc5b5fac4936d51a5dff656c281 Mon Sep 17 00:00:00 2001
From:
Date:
Subject: [PATCH 2/2] add time ms

---
 src/server.cr | 4 +++-
 1 file changed, 3 insertions(+), 1 deletion(-)

diff --git a/src/server.cr b/src/server.cr
index 2fad0ea..b08ae32 100644
--- a/src/server.cr
+++ b/src/server.cr
@@ -1,5 +1,6 @@
 require "logger"
 require "http/server"
+require "time"

 log = Logger.new(STDOUT)
 log.level = Logger::DEBUG
@@ -24,8 +25,9 @@ server = HTTP::Server.new(
     HTTP::CompressHandler.new,
     ]) do |context|
       if context.request.path == "/crystal" || context.request.path == "/crystal/"
+        epoch_ms = Time.now.epoch_ms
         context.response.content_type = "text/plain"
-        context.response.print "Crystal backend: Hello! from #{az_message} commit #{code_hash}"
+        context.response.print "Crystal backend: Hello! from #{az_message} commit #{code_hash} at #{epoch_ms}"
       elsif context.request.path == "/health"
         context.response.content_type = "text/plain"
         context.response.print "Healthy!"
--
2.22.0


EOF
```

* First, take a look at what changes are in the patch.

```bash
git apply --stat ecsdemo-crystal/add_time_ms.patch
```

* Run **git apply** to apply the patch.

```bash
sh -c 'cd ~/environment/ecsdemo-crystal && git apply add_time_ms.patch'
```

* Build the container

```bash
CRYSTAL_ECR_REPO=$(jq < cfn-output.json -r '.CrystalEcrRepo')

$(aws ecr get-login --no-include-email)

docker build -t crystal-service ecsdemo-crystal
docker tag crystal-service:latest $CRYSTAL_ECR_REPO:epoch
docker push $CRYSTAL_ECR_REPO:epoch
```
