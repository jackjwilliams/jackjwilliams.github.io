---
layout: post
published: false
author: jackjwilliams
title: Hangfire Windows Service
date: "2016-02-25 23:30"
comments: true
category: "ASP.NET 4.5"
tags: 
  - Hangfire
  - ASP.NET 4.5
  - MVC 5
subtitle: Pains of converting Hangfire to a Windows Service
modified: ""
mathjax: false
featured: false
---


I'm trying to keep these posts more focused and to the point, here we go ...

## Problem Statement

The problem with simply stuffing your Hangfire service into a Windows Service (maybe using Topshelf) is that beforehand
everything is run on the ASP.NET runtime. So some of the "Startup" stuff that comes with ASP.NET MVC doesn't necessarily happen
as you would expect. This costed me HOURS of debugging. Lets get to it.

### Problem #1: No IUserTokenProvider is registered.

NOTE: This is only when you need to do things like send out user registration tokens or password reset tokens in the background.
 
Out of the box, your UserManager.UserTokenProvider setup might look something like this:
{% highlight c# %}
var dataProtectionProvider = options.DataProtectionProvider;
if (dataProtectionProvider != null)
{
    manager.UserTokenProvider = 
        new DataProtectorTokenProvider<ApplicationUser>(
            dataProtectionProvider.Create("ASP.NET Identity")
        );
}
{% endhighlight %}

Our setup was a little different, in a partial Startup class we get and set the DataProtectionProvider off of the IAppBuilder,
then later check for it and set it similarly.

{% highlight c# %}
if (Startup.DataProtectionProvider != null)
{
    manager.UserTokenProvider = 
        new DataProtectorTokenProvider<ApplicationUser>(
            Startup.DataProtectionProvider.Create("ASP.NET Identity")
        );
}
{% endhighlight %}

But now that Hangfire is in it's own project and it's own Windows Service - this never happened (hence the error **No IUserTokenProvider is registered**). We have a lot of user creations / modifications
that we offload to the background.

**The fix?**

Use your own.

{% highlight c# %}
var provider = new DpapiDataProtectionProvider("MyApplicationName");
UserManager.UserTokenProvider = new DataProtectorTokenProvider<ApplicationUser>(provider.Create("UserToken"));
{% endhighlight %}

Reference: [Stackoverflow](http://stackoverflow.com/questions/22629936/no-iusertokenprovider-is-registered)

### Problem #2: SignalR

I like to provide the user with nice feedback and SignalR works great for giving users feedback and notifications for long running processes.

When running Hangfire in process with your ASP.NET application and injecting hubs into services using your favorite IoC, everything plays 
together nicely. When you move it to a Windows service - Hangfire doens't have the right context. This makes sense after thinking about it,
but at 2 A.M. it didn't make much sense.

**The fix?**

[Scaleout in SignalR](http://www.asp.net/signalr/overview/performance/scaleout-in-signalr)

I won't go into much detail as you will need to pick your scale out method, but I will tell you the basics of what I had to do to get going.

I chose Scaleout with SQL Server. In both your ASP.NET MVC Startup code and your Hangfire Server process code you have to tell which server to use.

Add this line in both projects after installing the NuGet package to both.

#### ASP.NET MVC
{% highlight c# %}
GlobalHost.DependencyResolver.UseSqlServer(sqlConnectionString);
app.MapSignalrR();
{% endhighlight %}

#### Hangfire Service Project
{% highlight c# %}
GlobalHost.DependencyResolver.UseSqlServer(sqlConnectionString);
{% endhighlight %}

For now I'm using the same database that my project is in simply because I don't need to scale that far out (yet). When ready I 
can update my cloud infrastructure, add a  (I database and modify this connection string. Pretty sweet!


### Problem #3: Custom Razor Templating

I have a custom Razor templating system (I can't remember which one - they're all fairly similar).

See [RazorEngine](https://github.com/Antaris/RazorEngine) or [Haacked Blog](http://haacked.com/archive/2011/08/01/text-templating-using-razor-the-easy-way.aspx/)

I use these a lot to create emails and other things. When these emails got pushed off to a hangfire background Job, I started getting exceptions
that the .cshtml file couldn't be found. 

**The fix?**

Open the properties of each .cshtml (or shift click them all), set to Build Action = Content and Copy to Output Directory = Copy Always.

### Problem #4: Dependencies, Dependencies, Dependencies

Did I mention dependencies?

Yeah, they were missing. Every. Where. Every which way and almost every Background job that ran was missing one. I'm guessing you know why ...
The hangfire service project needs references to all dependencies that will be called. One specifically I can think of is the SendGrid.SmtpApi. Before moving
to a Windows Service, all the dependencies were in with the ASP.NET project so it worked fine.

**The fix?**

Open Hangfire Server project, References > Add all missing references. This took awhile.

I hope this helps somebody because I was about ready to pull my hair out!