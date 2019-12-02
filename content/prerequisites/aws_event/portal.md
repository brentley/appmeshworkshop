---
title: "AWS Workshop Portal"
chapter: false
weight: 20
---

### Login to AWS Workshop Portal

This workshop creates an AWS acccount and a Cloud9 environment. You will need the **Participant Hash** provided upon entry, and your email address to track your unique session.

Connect to the portal by clicking the button or browsing to [https://dashboard.eventengine.run/](https://dashboard.eventengine.run/). The following screen shows up.

![Event Engine](/images/event-engine-initial-screen.png)

Enter the provided hash in the text box. The button on the bottom right corner changes to **Accept Terms & Login**. Click on that button to continue.

![Event Engine Dashboard](/images/event-engine-dashboard.png)

Click on **AWS Console** on dashboard.

![Event Engine AWS Console](/images/event-engine-aws-console.png)

Take the defaults and click on **Open AWS Console**. This will open AWS Console in a new browser tab.


{{% notice info %}}
If you are running this workshop at an AWS Event, someone might have warmed your account for you. This means that you might already have the baseline infrastructure created and ready. Follow the next step to confirm if this is the case.
{{% /notice %}}

After logging in to your AWS account, click on the following link to access the Cloud9 console: 

https://console.aws.amazon.com/cloud9/home

When in the Cloud9 console, check if there is a Cloud9 environment with the name `Project-appmesh-workshop` created. If so, click on `Open IDE` button to access the Cloud9 instance and continue the workshop from the chapter [**Update IAM Settings for your workspace**](/prerequisites/workspaceiam/)

If the Cloud9 environment is not present, head straight to [**Create a Workspace**](/prerequisites/workspace/).