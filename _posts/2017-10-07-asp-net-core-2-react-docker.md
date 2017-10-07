---
layout: post
published: true
author: Jack Williams
subtitle: 
category: personal
mathjax: false
featured: true
comments: true
title: ASP.NET Core 2 + React + Docker
---
## ASP.NET Core 2 + React Starter + Docker

I'll keep this post short, to serve as a reminder of how to build a production image of ASP.NET Core 2 + React + Docker.

For some reason, the React + Web API starter template in Visual Studio 2017 doesn't include docker support from the File > New Project menu. That night I never could find out how to add it (context menus didn't have "Add Docker Support" - probably due to late night coding). Later I found it, for whatever reason, and added it.

I have a two project solution currently:

src/PROJECT.Web

src/PROJECT.Core

With their respective .sln file

src/PROJECT.sln

The default Dockerfile location didn't work for me, as PROJECT.Web depended on PROJECT.Core.

This required significant modifications to the Dockerfile, as well as modifications to build/pack the react app.

1. Move the Dockerfile up to your root src folder or higher
2. Dockerfile contents (this will build + package a production image)

{% highlight docker %}
#Build
FROM microsoft/aspnetcore-build:1.0-2.0 AS build-env
COPY src /app
WORKDIR /app
RUN ["dotnet", "restore", "PROJECT.sln"]
RUN set -ex; \
	if ! command -v gpg > /dev/null; then \
		apt-get update; \
		apt-get install -y --no-install-recommends \
			gnupg2 \
			dirmngr \
		; \
		rm -rf /var/lib/apt/lists/*; \
	fi && curl -sL https://deb.nodesource.com/setup_8.x | bash - && apt-get update && apt-get install -y build-essential nodejs

# copy everything else and build
COPY . ./
RUN dotnet publish -c Release -o out PROJECT.sln

#Runtime 
FROM microsoft/aspnetcore:2.0.0

WORKDIR /app
COPY --from=build-env /app/PROJECT.Web/out ./
ENTRYPOINT ["dotnet", "PROJECT.Web.dll"]
{% endhighlight %}

3. Now run `docker build -t webapp .` in your Dockerfile location
