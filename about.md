---
layout: default
---
<div class="home">
  <div id="bio">
    <h2>Who am I?</h2>
    <p>Hi, my name is Fernando. A typical day for me involves hacking or studying and a good amount of time in the kitchen. My history within the IT world started when I was 14, when internet was a privilege of the insomniac. Every day I'd have to wait until 00:00 to connect with my wonderful <b>56kbps</b> dial-up modem, back when we only paid a single pulse to use the internet. These were the days of mIRC scripting: writing a lot of addons, bots, and hacking (un)protected channels for fun. Since then I've been a consumer of Information and Network Security texts. Being tech savvy made me the go-to maintenance guy for my family, no matter the subject: computers, cellphones, commercial software, and so on. That evolved into something that made me some money early in life. But old bricked/lost/scrapped pieces of hardware were piling up fast in my house, and I realized my earnings weren't enough to justify the stress and the clutter. That is when I decided to grab my first freelance job as a programmer.</p>

    <h2>What I do now</h2>
    <p>I'm an SRE specialised in observability: Grafana, Prometheus, Loki, SLOs, and the alerting layer sitting on top of all of it. The problem I keep circling back to is the one nobody puts on a dashboard: a monitoring stack that is technically green while the service underneath it degrades. Rules that never fire because the data never arrived. Grouping that compresses away the one dimension that mattered. Panels that answer a question nobody is asking at 3am.</p>
    <p>I write about that here, from things I've actually taken apart. Every example is generic by design: invented hostnames, invented service names, numbers chosen to show a shape rather than report one.</p>
    <p>If your alerts are green and you don't trust them, I'll look at them for free — <a href="{{ '/audit/' | relative_url }}">a scoped audit of your alert noise</a>, no obligation.</p>
    <p><a class="btn btn-primary" href="{{ '/audit/' | relative_url }}">Get your free alert audit</a></p>
  </div>

  {% include newsletter.html id="about" tag="about-page" %}
</div>
