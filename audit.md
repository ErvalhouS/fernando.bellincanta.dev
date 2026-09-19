---
layout: default
title: Free observability audit
description: I look at your alert noise and hand back a short fix list. No obligation.
permalink: /audit/
---

<div class="audit" markdown="1">

<p class="lede">Are your Grafana and Prometheus alerts all green while the service quietly degrades? Most alerting stacks aren't broken — they're desensitized. Send me your stack and I'll come back with the handful of alerts that actually matter, plus one thing you almost certainly have wrong. Around two hours of my time, free. If it's useful we can talk about going deeper; if not, you keep the analysis.</p>

<!--
  Intake form: a pre-filled mailto, deliberately. No third-party form service,
  no JS, no backend, nothing to break on a static site.
  OPTIONAL PLACEHOLDER: to book calls instead, replace both hrefs below with
  your Calendly URL, e.g. href="https://calendly.com/<your-handle>/audit".
-->
<p><a class="btn btn-primary" href="mailto:{{ site.email }}?subject=Free%20alert%20audit&body=1.%20Name%20%2B%20company%3A%0A%0A2.%20Stack%20%E2%80%94%20Grafana%3F%20Prometheus%3F%20Loki%3F%20SLOs%20defined%3F%20Self-hosted%20or%20Grafana%20Cloud%3F%0A%0A3.%20Roughly%20how%20many%20alerts%20does%20on-call%20receive%20per%20week%3F%0A%0A4.%20Current%20state%2C%20one%20sentence%3A%0A%0A5.%20Link%20to%20a%20public%20dashboard%20(optional)%3A%0A">Get your free alert audit</a></p>

## What I need from you

Five answers. They take about three minutes to write, and they're what makes the audit worth anything:

1. **Name and company.**
2. **Your stack.** Grafana? Prometheus? Loki? Are SLOs defined? Self-hosted or Grafana Cloud?
3. **Roughly how many alerts does on-call receive per week?** An estimate is fine. This is the single most useful number you can give me.
4. **Current state, in one sentence.** "We're firefighting." / "We don't know what to monitor." / "Everything is green but things still break."
5. **A link to a public dashboard** — optional, and it unlocks an extra 30 minutes of free review.

If your stack has no Grafana, no Prometheus and no SLOs, this audit won't help you, and I'll tell you that instead of wasting your afternoon.

## What you get

Two to three hours of work, delivered as a short written analysis:

- **A prioritized list of the 3–5 alerts that actually matter** in your stack — the name, why it matters, and what ignoring it costs you.
- **One sharp finding, verified.** Typical shapes: an alert that stays green because `no_data` was never configured; a rule evaluating against a log store the logs never reached; a datasource that silently resolves to the wrong backend; a dashboard panel whose query no longer matches the label it claims to group by.
- **A real count:** N alerts per week → N that deserve a human.

## What you don't get

This part is deliberate, and I'd rather be blunt about it up front:

- I don't apply the fix. The finding is stated, not implemented.
- I don't hand over the full noise-reduction plan.
- I don't touch anything in production.

## What happens after

Usually the audit surfaces two problems: one I can describe in a paragraph, and one that needs real debugging to confirm its actual impact. The second is the work I charge for — a fixed-scope engagement that ends with the root cause fixed, explicit `no_data` handling, corrected rules, and an alerting plan your on-call will actually run.

No pressure either way. If the free analysis is all you wanted, it's yours, and you owe me nothing.

<p><a class="btn btn-primary" href="mailto:{{ site.email }}?subject=Free%20alert%20audit&body=1.%20Name%20%2B%20company%3A%0A%0A2.%20Stack%3A%0A%0A3.%20Alerts%20per%20week%3A%0A%0A4.%20Current%20state%2C%20one%20sentence%3A%0A%0A5.%20Public%20dashboard%20link%20(optional)%3A%0A">Send me your stack</a></p>

{% include newsletter.html id="audit" tag="audit-page" heading="Not ready for an audit?" blurb="Take the writing instead. Short letters on observability, alerting and SLOs — the same material the audit comes out of." %}

</div>
