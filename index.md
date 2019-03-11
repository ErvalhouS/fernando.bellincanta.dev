---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
---
<link rel="stylesheet" type="text/css" href="{{ '/assets/custom.css' | relative_url }}">
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
          {%- if site.show_excerpts -%}
            {{ post.excerpt }}
          {%- endif -%}
        </div>
        <span class="post-meta">{{ post.date | date: date_format }}</span>
    </li>
    {%- endfor -%}
  </ul>

  <p class="feed-subscribe">
    <a href="{{ "/feed.xml" | relative_url }}" style="color: #f26522;">
      <svg class="svg-icon" style="height: 16px; width: 16px; fill: currentColor;">
        <use xlink:href="{{ '/assets/social-icons.svg#rss' | relative_url }}"></use>
      </svg>&nbsp;Subscribe
    </a>
  </p>
  {%- endif -%}

</div>
