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

## Solution

Create a new file called restrict_non_usa_ip_address.conf in the C:\Program Files\ModSecurity IIS\ directory and modify the file to look like this:

```
SecGeoLookupDb GeoLiteCity.dat
SecRule REMOTE_ADDR "@geoLookup" "id:'992210',phase:1,t:none,pass,nolog"
SecRule GEO:COUNTRY_CODE3 "!@streq USA" "id:'992211',phase:1,t:none,log,deny,msg:'Client IP not from USA'"
```

Next you need to download the GeoLiteCity data file, which has lookups to determine what IP's are in what Countries. The advantage to this approach is this is a data file and requires no additional DNS / HTTP calls to external servers. The file can be found [here](http://geolite.maxmind.com/download/geoip/database/GeoLiteCity.dat.gz).

Extract it to the C:\Program Files\ModSecurity IIS\ so that the GeoLiteCity.dat file is there. Line one of our new rule conf file (SetGeoLookupDb GeoLiteCity.dat) uses this data file.

Now modify your C:\Program Files\ModSecurity IIS\modsecurity_iis.conf file to include our new rule (I put it near the top to block any other rules from having to run).

```
Include restrict_non_usa_ip_address.conf
```

## Special Note For AWS / Load Balancers
If the IP that you typically see is your Load Balancers IP (as it was for me), you need to modify your restrict_non_usa_ip_address.conf to use the X-FORWARDED-FOR header instead of REMOTE_ADDR. In my case, AWS Load Balancers ALWAYS forward this. YMMV. See below.

```
SecGeoLookupDb GeoLiteCity.dat
SecRule REQUEST_HEADERS:X-FORWARDED-FOR "@geoLookup" "id:'992210',phase:1,t:none,pass,nolog"
SecRule GEO:COUNTRY_CODE3 "!@streq USA" "id:'992211',phase:1,t:none,log,deny,msg:'Client IP not from USA'"
```

That should be it! If you want more detailed logging, open modsecurity.conf in the root directory, find the lines that reference SecDebugLog and SecDebugLogLevel and set them, something like below.

SecDebugLog c:\inetpub\temp\debug.log
SecDebugLogLevel 9

Log Level 9 is HIGHLY VERBOSE and will generate 100's of MB's in a short time for an active (or inactive) site. You can search this file for the rule number "992211" to see lookups happening.