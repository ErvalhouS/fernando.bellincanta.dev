---
layout: default
---
<div class="home">
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
</div>
