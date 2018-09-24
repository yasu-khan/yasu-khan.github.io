---
layout: page
title: Resources
permalink: /Resources/
---
<br>
<div align="center"> A list of curate resources I have. </div>

<div>
{% for resources in site.resources reversed %}
  <br>
  {% if resources.date > site.time %}
    {% if forloop.first %}
      <h2 class="resources-section" id="upcoming">Upcoming</h2>
    {% endif %}
  {% else %}
    {% assign currentyear = resources.date | date: "%Y" %}
    {% if currentyear != previousyear %}
      <h2 class="resources-section" id="y{{ resources.date | date: "%Y"}}">{{ currentyear }}</h2>
      {% assign previousyear = currentyear %}
    {% endif %}
  {% endif %}


    <div class="resources">
      <span class="post-title"><strong><big> {{ resources.title }} </big></strong></span>
      {% if resources.subtitle %}
      <span class="post-date resources-subtitle"> - {{ resources.subtitle }} </span>
      {% endif %}
      <br>

      <i class="fa fa-comments-o" aria-hidden="true"></i>
      {% if resources.event-url %}
      <a href="{{ resources.event-url }}"
       title="{% if resources.event-fulltitle %}{{ resources.event-fulltitle }}{% else %}{{ resources.event }}{% endif %}
{{ resources.event-url }}">
      {% endif %}

      {{ resources.event }}{% if resources.event-fulltitle %}: {{ resources.event-fulltitle }}{% endif %}
      {% if resources.event-url %}</a>{% endif %}
      <br>

      <span class="location">
        <i class="fa fa-map-marker" aria-hidden="true"></i>
        {{ resources.location }}
      </span>
      <br>

      <i class="fa fa-calendar" aria-hidden="true"></i> {{ resources.date | date: "%B %-d, %Y" }}
      <br>

      {% if resources.slides %}
        <span class="resources-resource">
          <i class="fa fa-file-pdf-o" aria-hidden="true"></i>
          {% if resources.slides contains ".pdf" %}
            <a href="{{ site.pdfs }}/{{ resources.slides }}">
          {% else %}
            <a href="{{ resources.slides }}">
          {% endif %}
          Slides</a>
        </span>
      {% endif %}
      
      {% if resources.video %}
        &nbsp; &nbsp;
        <span class="resources-resource"><i class="fa fa-file-video-o" aria-hidden="true"></i> <a href="{{ resources.video }}">Video</a></span>
      {% endif %}
      
      {% if resources.post %}
        &nbsp; &nbsp;
        <span class="resources-resource">
          <i class="fa fa-file-text-o" aria-hidden="true"></i>
          <a href="{{ site.url }}{{ resources.post }}">Post</a>
        </span>
      {% endif %}
  
      {% if resources.featured %}
        &nbsp; &nbsp;
        <span class="resources-resource"><i class="fa fa-file-text-o" aria-hidden="true"></i> <a href="{{ resources.featured }}">Featured</a></span>
      {% endif %}
    </div>
{% endfor %}
</div>

