---
layout: post
published: false
author: author
subtitle: With an HttpHandler
modified: ""
category: "C#"
mathjax: false
featured: false
comments: false
title: Secured Image Server Using ImageResizer.NET
---


## A New Post

I've been struggling with an issue lately of securing and resizing user uploaded images. I started looking at [Cloudinary](http://cloudinary.com/ "Cloudinary"), but quickly realized their service is mostly for publicly viewable media. They do have options for kind-of-secured (read [here](http://cloudinary.com/blog/delivering_all_your_websites_images_through_a_cdn) and [here](http://support.cloudinary.com/hc/en-us/articles/202519742-Can-I-allow-access-to-uploaded-images-only-to-authenticated-users-)), but it is much more costly and still did not really give us what we wanted. I could throw them in a folder and require that the request be authenticated to get to the folder, but we are talking about a LOT of images. This is on an Amazon EC2 instance - not exactly the place to store user uploaded images. Especially my users - they want to upload HUGE FILES (this is in the gov space, and some photos can be 30mb for certain types). 
