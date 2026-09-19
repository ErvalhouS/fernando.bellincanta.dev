---
layout: default
title: Fernando Bellincanta
description: SRE working on observability and alerting for voice platforms at carrier scale. Grafana, Prometheus, Loki, SLOs.
permalink: /about/
---
<div class="home">
  <div id="bio">
    <h2>What I do</h2>
    <p>I'm an SRE specialised in observability: Grafana, Prometheus, Loki, SLOs, and the alerting layer sitting on top of all of it. Most of that work happens on voice and video platforms, where a degradation is measured in packets and milliseconds and rarely looks like an outage from the outside.</p>
    <p>The problem I keep circling back to is the one nobody puts on a dashboard: a monitoring stack that is technically green while the service underneath it degrades. Rules that never fire because the data never arrived. Grouping that compresses away the one dimension that mattered. Panels that answer a question nobody is asking at 3am.</p>

    <h2>What I write here</h2>
    <p>Things I've actually taken apart, with the failure mode stated plainly enough that you can go check your own stack against it. Every example is generic by design: invented hostnames, invented service names, numbers chosen to show a shape rather than report one.</p>
    <p>If your alerts are green and you don't trust them, I'll look at them for free: <a href="{{ '/audit/' | relative_url }}">a scoped audit of your alert noise</a>, no obligation.</p>
    <p><a class="btn btn-primary" href="{{ '/audit/' | relative_url }}">Get your free alert audit</a></p>

    <h2>How I got here</h2>
    <p>I started at 14, when internet was a privilege of the insomniac and I had to wait until midnight to connect with a 56kbps dial-up modem for the price of a single pulse. Those were the mIRC years: writing addons and bots, hacking (un)protected channels for fun, and reading every network security text I could find. Being the family's fix-it guy paid for a while, until the pile of bricked hardware in my house outgrew what it earned me, and I took my first freelance job as a programmer instead.</p>
    <p>A typical day still involves hacking or studying, and a good amount of time in the kitchen.</p>
  </div>

  {% include newsletter.html id="about" tag="about-page" %}
</div>
