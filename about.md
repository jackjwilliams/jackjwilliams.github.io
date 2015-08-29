---
layout: page
permalink: /about/index.html
title: Amey Jadiye
tags: 
  - Amey
  - Jadiye
chart: true
published: true
---



{% assign total_words = 0 %}
{% assign total_readtime = 0 %}
{% assign featuredcount = 0 %}
{% assign statuscount = 0 %}

{% for post in site.posts %}
    {% assign post_words = post.content | strip_html | number_of_words %}
    {% assign readtime = post_words | append: '.0' | divided_by:200 %}
    {% assign total_words = total_words | plus: post_words %}
    {% assign total_readtime = total_readtime | plus: readtime %}
    {% if post.featured %}
    {% assign featuredcount = featuredcount | plus: 1 %}
    {% endif %}
{% endfor %}

Thanks for taking the time to check out my blog - this is my first! I recently went to a developer conference and learned it is about time I share some knowledge and quit taking so much. This is my attempt to do that. 

I'm a developer by day and well, by night too. 

My primary focus over the last few years has been C# .NET MVC (2/3/4/5), Android and a sprinkling of Ruby on Rails.

Current stats:

{{ site.posts | size }} posts in {{ site.categories | size }} categories which combined have {{ total_words }} words. 

{% if featuredcount != 0 %}There are <a href="{{ site.url }}/featured">{{ featuredcount }} featured posts</a>, you should definitely check those out.{% endif %}
