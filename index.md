---
layout: page
title: Home
---
{% include JB/setup %}

I am Anshu Avinash, currently pursuing a BT-MT dual degree in [Computer Science and Engineering](http://cse.iitk.ac.in) from [IIT Kanpur](http://iitk.ac.in). Here is a list of all my posts till now:  

<ul>
{% for post in site.posts %}
  <li><a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %} 
</ul>
