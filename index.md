---
layout: default
---
<div class="home">
  <div class="cta-hero">
    <h2>Free observability audit</h2>
    <p>Green alerts you don't trust? Send me your stack and I'll come back with the 3–5 alerts that actually matter, plus one thing you almost certainly have wrong. Around two hours of my time, free, no obligation.</p>
    <p><a class="btn btn-primary" href="{{ '/audit/' | relative_url }}">Get your free alert audit</a></p>
  </div>

  {%- if site.posts.size > 0 -%}
  <h2 class="post-list-heading">Posts</h2>
  <ul class="post-list">
    {%- for post in site.posts -%}
    <li>
      {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
        <h3>
          <a class="post-link" href="{{ post.url | relative_url }}">
            {{ post.title | escape }}
          </a>
        </h3>
        <div class="excerpt">
          {{ post.description }}
        </div>
        <span class="post-meta">{{ post.date | date: date_format }}</span>
    </li>
    {%- endfor -%}
  </ul>
  {%- endif -%}

  {% include newsletter.html id="home" tag="home-page" %}
</div>
