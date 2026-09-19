---
layout: default
title: The newsletter
description: Short letters on observability, alerting and SLOs. What deserves to page a human, and what only looks like it does.
permalink: /newsletter/
---

<div class="audit" markdown="1">

<p class="lede">Most weeks I send one short letter about the alerting layer: a failure mode I hit, why the stack stayed green through it, and the change that would have caught it. No roundups, no tool news, no "10 best Grafana plugins".</p>

## What you get

- **One failure mode per letter**, stated plainly enough that you can go check your own stack against it the same afternoon.
- **The query, rule or config that fixes it**, not a gesture at one.
- **Generic by design.** Invented hostnames, invented service names, numbers chosen to show a shape rather than report one. Nothing here comes from anyone's production but the shape of the problem is real.

Recent subjects: alert payloads that report `active: []` while the metrics pipeline is down, per-path SLOs that site averages compress away, `no_data` handling nobody configured.

{% include newsletter.html id="newsletter-page" tag="newsletter-page" heading="Subscribe" blurb="One letter most weeks, straight to your inbox." %}

<p><a href="https://buttondown.com/{{ site.buttondown_username }}/archive/">Read the archive first</a> if you'd rather see the shape of it before handing over an address.</p>

## If you'd rather I just look at yours

The letters come out of the same work as the [free alert audit]({{ '/audit/' | relative_url }}): you send me your stack, I come back with the handful of alerts that actually matter and one thing you almost certainly have wrong.

<p><a class="btn btn-primary" href="{{ '/audit/' | relative_url }}">Get your free alert audit</a></p>

</div>
