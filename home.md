---
layout: page
title: Posts
sidebar_link: true
---

<ul style="list-style-type: none;">
{% for post in site.posts %}
<li style = "border-bottom: 2px solid #999; margin-bottom: 10px;">
    <a href="{{ post.url | prepend: site.baseurl  }}" style="text-decoration: none; font-size: 2rem; color: black;">  
      <span style="text-decoration: underline;">{{ post.title }}</span>
    </a>
    <br>
    <br>
     <p class="message"> <a href="{{ post.url | prepend: site.baseurl  }}" style="text-decoration: none; color: black;">{{ post.subtitle }} </a></p>
    <br>
</li>
{% endfor %}
</ul>

