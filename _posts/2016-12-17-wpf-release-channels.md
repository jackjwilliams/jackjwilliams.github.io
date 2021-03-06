---
layout: post
published: false
author: Jack Williams
subtitle: with Squirrel & Octopus Deploy
mathjax: false
featured: false
comments: false
title: WPF Release Channels
modified: '2016-12-17'
category: 'C#'
---
## The Problem

Recently I started doing WPF development and learned that deployments are not as easy to handle as web applications. With a web application, using Teamcity & Octopus Deploy, it's easy to build, deploy and promote between environments. Because it's "hosted", in one spot, on a web server. But what about a WPF application for a large company with thousands of employees?

How do you "promote" a WPF from "Nightly" to "Beta" when hundreds or thousands of users have it installed on their desktop?

Damn good question. It's not easy, I'll tell you that.

I've been working with TeamCity about 5 years and Octopus Deploy a couple of years. They are amazing tools! Alright let's look at some potential solutions to our problem.

### ClickOnce

ClickOnce has some nice publishing capabilities. While I won't go into all the details of ClickOnce deployments, you can read about it's features [here](https://msdn.microsoft.com/en-us/library/t71a733d.aspx).

I tried to integrate ClickOnce with Octopus following some of the suggestions [here](http://help.octopusdeploy.com/discussions/questions/2434-deploying-clickonce-application), but failed.

But then the big question remains: How do you "promote" between channels? When beta is done, how does it go to production?

### Squirrel.Windows

Here is an interesting project. Open source, pretty active. It's motto is "Squirrel: It's like ClickOnce but Works™". Ha!

It has a client API to hook into update events (for "Splash-Screen-Update-Progress-Bar-Thingees") - yay we can tell our users exactly what's happening!

It supports delta update packages

It supports "channels" like Dev/Beta/Release.

I'm sold! Let's get started ...

But first let me give props to Chase Florell, who got me going with [this](https://chaseflorell.github.io/devops/2015/08/07/deploying-squirrel.windows-apps-with-octopus-deploy/) amazing post! This is mostly all of his suggestions, with a few tweaks for promoting between channels, desktop icon names, etc...

### Issues I Ran Into

* Desktop Icon Names and how to change them (MyClient, MyClient Beta, MyClient Nightly)
* Building once, deploying everywhere - NOT
* Lots of Octopus Channels, Environments and Lifecycles (see [this](http://help.octopusdeploy.com/discussions/questions/9063-build-once-deploy-everywhere-with-exes) Octopus forum post I made)


### General Process

1. Build WPF application
