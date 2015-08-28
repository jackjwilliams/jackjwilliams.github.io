---
published: false
---

## Hangfire + MVC 5 Goodness
I recently attended my first developer conference - [Code on the Beach 2015](https://www.codeonthebeach.com/) - and I must say - it was amazing! Among all the great talks, one caught my attention by Elijah Manor (Keynote) where he talks about [growing developers](https://www.codeonthebeach.com/cotb2015/session/3124/growing-developers). One thing I haven't done much of as a developer is share. I take all the time (reading blogs, researching answers to problems, stackoverflow, etc...), then I build - and leave it at that. This is my first attempt at giving.

I've been working on a project lately where some processes were getting bogged down due to some long running commands / queries. Rightly so, as the first iteration we were trying to get the concept up and running for the client - no optimization. When it came time to put some tasks in the background - we decided to go with [Hangfire](http://hangfire.io/). This post will focus mainly on the setting up and implementation of Hangfire with SimpleInjector and MVC 5.

### Database Setup with Local / Staging / Develop / Production Environments
One issue we had upon first deploying Hangfire was how to automate the creation of the Hangfire databases. We had so many environments, and did not want to re-use the same database for each one - and definitely did not want to have to manually create each database. 

#### Local
The [documentation](http://docs.hangfire.io/en/latest/configuration/using-sql-server.html) states that

> SQL Server objects are being installed **automatically** from the SqlServerStorage constructor by executing statements described in the Install.sql file (which is located under the tools folder in the NuGet package). Which contains the migration script, so new versions of Hangfire with schema changes can be installed seamlessly, without your intervention.

But this never happened for me - locally we use a (LocalDb) file, and if it didn't exist Hangfire barfed:
{% highlight c# %}
An attempt to attach an auto-named database for file C:\....\App_Data\Hangfire.mdf failed. A database with the same name exists, or specified file cannot be opened, or it is located on UNC share.
{% endhighlight %}