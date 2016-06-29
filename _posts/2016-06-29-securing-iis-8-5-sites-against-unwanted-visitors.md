---
layout: post
published: true
author: author
subtitle: Subtitle
mathjax: false
featured: false
comments: false
title: Securing IIS 8.5 Sites Against Unwanted Visitors
---
## I'm Being Attacked!

At around 4:00 PM, Tuesday, June 28'th my Raygun inbox started filling with 404 errors (lots and lots of them, neverending). My IIS logs looked something like this:
```
2016-06-29 08:17:02 10.X.X.X GET /assets/plugins/lightbox/Images/url - 443 - 10.X.X.X Mozilla/5.0+(X11;+Ubuntu;+Linux+x86_64;+rv:47.0)+Gecko/20100101+Firefox/47.0 - 302 0 2 0 212.83.40.238
2016-06-29 08:17:02 10.X.X.X GET /assets/plugins/lightbox/Images/urlrewriter - 443 - 10.X.X.X Mozilla/5.0+(X11;+Ubuntu;+Linux+x86_64;+rv:47.0)+Gecko/20100101+Firefox/47.0 - 302 0 2 15 212.83.40.238
2016-06-29 08:17:02 10.X.X.X GET /assets/plugins/lightbox/Images/urls - 443 - 10.X.X.X Mozilla/5.0+(X11;+Ubuntu;+Linux+x86_64;+rv:47.0)+Gecko/20100101+Firefox/47.0 - 302 0 2 0 212.83.40.238
2016-06-29 08:17:03 10.X.X.X GET /assets/plugins/lightbox/Images/US - 443 - 10.X.X.X Mozilla/5.0+(X11;+Ubuntu;+Linux+x86_64;+rv:47.0)+Gecko/20100101+Firefox/47.0 - 302 0 2 0 212.83.40.238
2016-06-29 08:17:03 10.X.X.X GET /assets/plugins/lightbox/Images/usa - 443 - 10.X.X.X Mozilla/5.0+(X11;+Ubuntu;+Linux+x86_64;+rv:47.0)+Gecko/20100101+Firefox/47.0 - 302 0 2 0 212.83.40.238
2016-06-29 08:17:03 10.X.X.X GET /assets/plugins/lightbox/Images/us - 443 - 10.X.X.X Mozilla/5.0+(X11;+Ubuntu;+Linux+x86_64;+rv:47.0)+Gecko/20100101+Firefox/47.0 - 302 0 2 0 212.83.40.238
2016-06-29 08:17:04 10.X.X.X GET /assets/plugins/lightbox/Images/user - 443 - 10.X.X.X Mozilla/5.0+(X11;+Ubuntu;+Linux+x86_64;+rv:47.0)+Gecko/20100101+Firefox/47.0 - 302 0 2 0 212.83.40.238
2016-06-29 08:17:04 10.X.X.X GET /assets/plugins/lightbox/Images/usage - 443 - 10.X.X.X Mozilla/5.0+(X11;+Ubuntu;+Linux+x86_64;+rv:47.0)+Gecko/20100101+Firefox/47.0 - 302 0 2 15 212.83.40.238
2016-06-29 08:17:04 10.X.X.X GET /assets/plugins/lightbox/Images/user_upload - 443 - 10.X.X.X Mozilla/5.0+(X11;+Ubuntu;+Linux+x86_64;+rv:47.0)+Gecko/20100101+Firefox/47.0 - 302 0 2 0 212.83.40.238
```

If it were just a few I would have chalked it up to some crawlers. But these were persistent, about 12 hours worth of pummeling our server trying to discover flaws / files in our system. Or maybe it was a DoS attack - I'm not really sure.

I did some research and found a nifty little HTTP Module called ModSecurity that works with IIS 8.5.

[Here](https://www.modsecurity.org/download.html "ModSecurity Download Page") is the download page.

You also need vcredist for VC++ 2013, and it can be downloaded [here](https://www.microsoft.com/en-us/download/details.aspx?id=40784). INSTALL THIS FIRST.

## Installation Notes - IMPORTANT
Once you install ModSecurity, it will immediately take affect on all sites! ModSecurity works with a set of rules, all stored in files in the C:\Program Files\ModSecurity IIS\ directory. Out of the box it gives you a set of OWASP base rules in the C:\Program Files\ModSecurity IIS\owasp_crs\base_rules directory. These are TOO RESTRICTIVE! I could not even log into my own site because it was stripping the POST values.

To fix: Open the C:\Program Files\ModSecurity IIS\modsecurity_iis.conf file and comment out the below line using a #.

#Include owasp_crs\base_rules\*.conf

Whew, now our site works again.

## The Source of the Problem
One thing I noticed right off the bat was that all of these requests were from outside of the United States. Given that the application I'm trying to protect is for Law Enforcement Agencies within the United States, I need one rule to ring them all: Block all IP's outside the US.

Let's create our new rule!




