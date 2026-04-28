---
layout: archive
permalink: /year-archive/
title: "Notes"
author_profile: true
---

Occasional writing on things I have built or figured out the hard way.

{% capture written_label %}'None'{% endcapture %}
{% for post in site.posts %}
{% capture year %}{{ post.date | date: '%Y' }}{% endcapture %}
{% if year != written_label %}
<h2>{{ year }}</h2>
{% capture written_label %}{{ year }}{% endcapture %}
{% endif %}
<p><a href="{{ post.url }}">{{ post.title }}</a><br><small>{{ post.date | date: "%B %d, %Y" }}</small></p>
{% endfor %}
