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
Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.
