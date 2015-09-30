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
title: Secured Image Server Using ImageResizer
---


## A New Post

I've been struggling with an issue lately of securing and resizing user uploaded images in Amazon S3. I started looking at [Cloudinary](http://cloudinary.com/ "Cloudinary"), but quickly realized their service is mostly for publicly viewable media. They do have options for kind-of-secured (read [here](http://cloudinary.com/blog/delivering_all_your_websites_images_through_a_cdn) and [here](http://support.cloudinary.com/hc/en-us/articles/202519742-Can-I-allow-access-to-uploaded-images-only-to-authenticated-users-)) images, but it is much more costly and still did not really give me what I wanted. I could throw them in a folder and require that the request be authenticated to get to the folder, but we are talking about a LOT of images. This is on an Amazon EC2 instance - not exactly the place to store user uploaded images. Especially my users - they want to upload HUGE FILES (this is in the gov space, and some photos can be 30mb for certain types). 

It became apparent that I will need to roll my own solution, and this is where I found [ImageResizer](http://imageresizing.net/). It will allow me to access my Amazon S3 private images, and transform them as needed (to thumbnail, auto-rotate, etc...) without having to generate signed, expiring urls every single time I need to access an image. 

But I don't want to do all this on my primary server. It does enough work on its own. This post is going to show how I setup an ImageResizer server, and a proxy HttpHandler on my primary server to only allow authenticated users in my current webapp access to a given image on my ImageResizer server.

Currently we have to do something like the following for every image that the user requests:

