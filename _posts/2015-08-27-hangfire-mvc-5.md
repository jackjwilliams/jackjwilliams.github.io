---
published: false
---

## Hangfire + MVC 5 Goodness
I recently attended my first developer conference - [Code on the Beach 2015](https://www.codeonthebeach.com/) - and I must say - it was amazing! Among all the great talks, one caught my attention by Elijah Manor (Keynote) where he talks about [growing developers](https://www.codeonthebeach.com/cotb2015/session/3124/growing-developers). One thing I haven't done much of as a developer is share. I take all the time (reading blogs, researching answers to problems, stackoverflow, etc...), then I build - and leave it at that. This is my first attempt at giving.

I've been working on a project lately where some processes were getting bogged down due to some long running commands / queries. Rightly so, as the first iteration we were trying to get the concept up and running for the client - no optimization. When it came time to put some tasks in the background - we decided to go with [Hangfire](http://hangfire.io/). This post will focus mainly on the setting up and implementation of Hangfire with SimpleInjector and MVC 5.

### Database with Staging / Develop / Production Environments
One issue we had upon first deployment was how to automate the creation of the Hangfire database. 