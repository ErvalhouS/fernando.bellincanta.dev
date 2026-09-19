---
layout: default
title: Free observability audit
description: I look at your alert noise and hand back a short fix list. No obligation.
permalink: /audit/
---

<div class="audit" markdown="1">

<p class="lede">Are your Grafana and Prometheus alerts all green while the service quietly degrades? Most alerting stacks aren't broken. They're desensitized. Send me your stack and I'll come back with the handful of alerts that actually matter, plus one thing you almost certainly have wrong. Around two hours of my time, free. If it's useful we can talk about going deeper; if not, you keep the analysis.</p>

<!--
  Intake goes to the Tally form in `audit_form_url` (_config.yml), which
  redirects to /audit/thanks/ on submit. The mailto below it is the fallback
  for anyone who would rather not touch a form.
  ?source= populates a hidden field named `source` in Tally, so every lead
  arrives knowing which button it came from. Top vs bottom tells you whether
  the page copy is doing any work.
-->
<p><a class="btn btn-primary" href="{{ site.audit_form_url }}?source=audit-top">Get your free alert audit</a></p>
<p class="cta-alt">or <a href="mailto:{{ site.email }}?subject=Free%20alert%20audit">email me directly</a> if you would rather not fill in a form.</p>

## What I need from you

Five answers. They take about three minutes to write, and they're what makes the audit worth anything:

1. **Name and company.**
2. **Your stack.** Grafana? Prometheus? Loki? Are SLOs defined? Self-hosted or Grafana Cloud?
3. **Roughly how many alerts does on-call receive per week?** An estimate is fine. This is the single most useful number you can give me.
4. **Current state, in one sentence.** "We're firefighting." / "We don't know what to monitor." / "Everything is green but things still break."
5. **A link to a public dashboard.** Optional, and it unlocks an extra 30 minutes of free review.

If your stack has no Grafana, no Prometheus and no SLOs, this audit won't help you, and I'll tell you that instead of wasting your afternoon.

## What you get

Two to three hours of work, delivered as a short written analysis:

- **A prioritized list of the 3–5 alerts that actually matter** in your stack: the name, why it matters, and what ignoring it costs you.
- **One sharp finding, verified.** Typical shapes: an alert that stays green because `no_data` was never configured; a rule evaluating against a log store the logs never reached; a datasource that silently resolves to the wrong backend; a dashboard panel whose query no longer matches the label it claims to group by.
- **A real count:** N alerts per week → N that deserve a human.

## What you don't get

This part is deliberate, and I'd rather be blunt about it up front:

- I don't apply the fix. The finding is stated, not implemented.
- I don't hand over the full noise-reduction plan.
- I don't touch anything in production.

## What happens after

Usually the audit surfaces two problems: one I can describe in a paragraph, and one that needs real debugging to confirm its actual impact. The second is the work I charge for: a fixed-scope engagement that ends with the root cause fixed, explicit `no_data` handling, corrected rules, and an alerting plan your on-call will actually run.

No pressure either way. If the free analysis is all you wanted, it's yours, and you owe me nothing.

<p><a class="btn btn-primary" href="{{ site.audit_form_url }}?source=audit-bottom">Send me your stack</a></p>
<p class="cta-alt">or <a href="mailto:{{ site.email }}?subject=Free%20alert%20audit">email me directly</a> if you would rather not fill in a form.</p>

{% include newsletter.html id="audit" tag="audit-page" heading="Not ready for an audit?" blurb="Take the writing instead. Short letters on observability, alerting and SLOs, the same material the audit comes out of." %}

</div>
