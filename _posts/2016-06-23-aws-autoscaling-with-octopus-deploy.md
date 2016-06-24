---
layout: post
published: false
author: Jack Williams
mathjax: false
featured: false
comments: false
title: AWS AutoScaling with Octopus Deploy
---
## Fear Not, It Can Be Done

Let me first give accolades to Jeffery Palermo. He did a bootcamp on Continuous Delivery at the Kansas City Developer Conferencce that I attended - it really made me re-think our CD process. It was his kindness and helpfulness to other developers and I that reminded me I need to give back as well. I've been a long time reader of his blogs and books, and overall fan. It was really cool to meet him in person and see how much of a helping spirit he has.

I've been meaning to blog about this but just haven't found the time. Now is as good a time as any, I just got done with day one of the KCDC.

### Background

On the project I'm currently working on I had to figure out how to setup autoscaling with Octopus Deploy - and it was no simple feat. There wasn't a ton of information on this topic. This post details the process I came up with to handle this.

### Process Outline

1. Create scripts to
	- Delete old, stale tentacles
    - Create a new tentacle
    - Assign the tentacle the appropriate roles
    - Assign the tentacle the appropriate environments
    - Push the latest Octopus Deployment to the new tentacle
2. Create an AWS Launch Configuration which
	- Spins up a new instance
    - Pulls in all needed scripts (from above) and executables
    - Execute the scripts
    
### What this post is NOT about.

- Setting up Autoscaling in AWS
- Setting up a Cloud Formation template

### Requirements

1. An understanding of Autoscaling groups and LaunchConfigurations in AWS
2. An understanding of CloudFormation in AWS
3. Patience
4. Patience

I mention patience twice because testing your scripts takes killing one instance and spinning up a new one a few times. This isn't always the case, but once you get them running it takes a few to make sure everything is on the up and up.

### Tips
1. To avoid the stop and restart instance cycle
	a. Get everything setup like you want, then do an initial scaling operation.
    b. If it doesn't work, RDP into your instance and GET them working manually.
    c. If it does work, REJOICE.
2. Log files are your friend, you can find them on the new instance in C:\cfn\log.
3. Turn OFF auto-rollback when your cloud fails to start (otherwise you can't look at the logs).



Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.
