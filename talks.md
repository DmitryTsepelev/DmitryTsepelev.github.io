---
layout: base
title: Talks
permalink: /talks/
---

<div class="home">
  {% assign talks = site.talks | sort: "date" | reverse %}

  {%- if talks.size > 0 -%}
    <div class="talks-grid">
      {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
      {% for talk in talks %}
      <a class="talk-card" href="{{ talk.url | relative_url }}">
        <div class="talk-card__header">
          <span class="post-meta">{{ talk.date | date: date_format }}</span>
          <span class="talk-card__conference">{{ talk.conference }}</span>
        </div>
        <h3 class="talk-card__title">{{ talk.title | escape }}</h3>
        {%- assign talk_excerpt = talk.content | strip_html | truncate: 120 -%}
        {%- if talk_excerpt.size > 0 -%}
          <p class="talk-card__description">{{ talk_excerpt }}</p>
        {%- endif -%}
        <div class="talk-card__footer">
          {%- if talk.youtube_link -%}
            <span class="talk-card__badge">Video</span>
          {%- endif -%}
          {%- if talk.slides_link -%}
            <span class="talk-card__badge">Slides</span>
          {%- endif -%}
        </div>
      </a>
      {%- endfor -%}
    </div>
  {%- endif -%}
</div>
