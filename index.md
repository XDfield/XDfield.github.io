---
layout: index
title: DoSun's Blog
---
# 这里是DoSun的个人博客!  
主要记录下平常学习过程中的一些笔记吧.  

{% for post in site.posts %}
<article class="post">
  {% if post.img %}
    <a class="post-thumbnail" style="background-image: url({{ site.img.link | append : post.img}})" href="{{post.url | prepend: site.baseurl}}"></a>
  {% else %}
  {% endif %}
  <div class="post-content">
    <h2 class="post-title"><a href="{{post.url | prepend: site.baseurl}}">{{post.title}}</a></h2>
    <p>{{ post.content | strip_html | truncate: 100 }}</p>
    <span class="post-date">{{post.date | date: '%Y, %b %d'}}</span>
    <!-- <span class="post-words">{% capture words %}{{ post.content | number_of_words }}{% endcapture %}{% unless words contains "-" %}{{ words | plus: 250 | divided_by: 250 | append: " minute read" }}{% endunless %}</span> -->
  </div>
</article>
{% endfor %}
