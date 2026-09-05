---
layout: post
title:  "The empty alert payload: why autonomous triage fails on representation, not reasoning"
description: "Two weeks of running an LLM-driven triage loop over a telecom alerting stack, and every wrong conclusion came from the summary layer instead of the model. What that says about the autonomous NOC."
date:   2026-09-05 09:00:00 -0300
categories: Concepts
permalink: /:categories/:title
---

The worst thirty seconds of my last two weeks were the ones where the alert payload said `active: []`.

Zero active alerts. Nothing firing. A quiet Saturday morning on a voice platform, which is exactly the kind of morning you want. Except that in the three hours before that payload was generated, the metrics stack on one of the clusters had stopped shipping blocks to long-term storage, and every scrape target in the DNS layer had gone missing. The alerting was fine. The alerts had fired, in the channel, with timestamps, all night. What was empty was the summary: the derived, compressed, machine-readable artifact that my triage loop actually read.

I spent the last two weeks building an automated triage loop over a telecom observability stack (alerts from SIP media nodes, regional RPC proxies, Kafka consumer groups, Kubernetes control planes) and running it every fifteen minutes. It produced a lot of correct conclusions and a handful of confidently wrong ones. Not one of the wrong ones was a reasoning failure. Every one was a **representation** failure: the summary the reasoning ran on had thrown away the bit that mattered.

That is in tension with how the autonomous-NOC pitch is currently framed. The 2026 vendor story is that agentic incident response is bottlenecked on model capability, and that a bigger model with a better correlation prompt closes the gap. My two weeks say the bottleneck is somewhere much less glamorous: the fidelity contract between your alerting system and whatever consumes it.

## 1. Grouping is compression, and compression is lossy

The first thing that bit me was alert grouping. Alertmanager groups by label set, and a group renders as one notification with a count: `[FIRING x 4]`. That is the correct product decision, since it is what stops a fifty-node flap from becoming fifty pages.

It also means **one line in your payload is N incidents**, and N is a moving number you have to actually read.

I had a rule for connectivity loss across an overlay network that would render as a single row. The row said one thing. The underlying group contained four distinct hosts across two sites. My summarizer, quite reasonably, emitted one object per notification, so downstream I "knew" about one host. When a second site joined the group, the row did not change shape at all. The count went from 2 to 4, buried in a title I was regex-stripping.

Worse: a group can partially resolve. Alertmanager will happily tell you that one instance in the group is not firing anymore while the group as a whole stays active. If you track state per rule rather than per instance, you will see a rule that has been "continuously firing for six hours" when in reality it has been a churn of different hosts entering and leaving, each for twenty minutes. Those are opposite incidents with the same summary.

The fix is not clever. It is to stop treating the notification as the unit and start treating the instance fingerprint as the unit:

```bash
# The unit of state is the fingerprint, not the rendered notification.
curl -s http://alertmanager.internal:9093/api/v2/alerts \
  | jq -r '
      map(select(.status.state == "active"))
      | map({
          fp:    .fingerprint,
          rule:  .labels.alertname,
          inst:  (.labels.instance // "GROUPED-NO-INSTANCE"),
          since: .startsAt
        })
      | sort_by(.rule, .inst)[]
      | [.rule, .inst, .since, .fp] | @tsv'
```

The `GROUPED-NO-INSTANCE` fallback is the important line. It works as a detector rather than as a default value. Any alert reaching your triage layer without an instance-level identity is an alert you cannot count, and you want that fact loud rather than papered over with a null.

## 2. "Resolved" is a field, and fields lie

For about a week I believed the alerting stack never resolved anything. Every payload I got showed `resolutions_seen: 0`. I built a whole mental model on it: durations are meaningless, everything stays open forever, rank by age of the last firing timestamp instead.

That model was wrong, and it cost me several triage rounds. The alerts did resolve. The resolution notifications were posted, visibly, in a channel, as messages with a different structure that my collector's parser skipped. The `resolved` field in my payload was not reporting reality. It was reporting my parser's coverage.

Then the same field failed the other way. I got objects marked `resolved` whose referenced timestamp pointed at a firing notification. The summarizer had matched the wrong end of a pair. So the field was simultaneously under-reporting real resolutions and over-reporting fake ones, and there was no way to tell from inside the payload.

The general lesson, which I now apply everywhere: **a derived status field is a claim about your pipeline, not about your infrastructure.** The only trustworthy resolution evidence is a state transition you observed with its own timestamp, an event rather than an attribute. If your triage layer cannot point at "this fingerprint was active at T1 and absent at T2," it does not know the alert resolved. It knows someone wrote `resolved: true`.

That reframing also kills a metric I had been leaning on hard. "This alert has been open for 27 hours" feels like a severity signal. It is not. In a stack where resolution detection is unreliable, duration measures how long nobody closed it, which is a statement about human attention. A genuinely stuck condition and a flapping condition nobody acknowledged produce the identical number.

## 3. Silence is the hardest state to represent

Here is the shape that nearly caught me twice.

A disk-utilisation alert on a media node had been climbing for days. It hit 99% and then it stopped firing. In every heuristic I had, "stopped firing" ranks below "still firing." Persistent alerts that never grow are noise; alerts that go quiet are resolved. Except a saturated metric that goes silent is not a metric that recovered. It is an exporter that died, or a host that did, and it is the single highest-severity state on the board rendered as absence.

Absence has no row. You cannot rank what is not in the list.

This is what a dead man's switch is for, and it is why every serious Prometheus deployment ships a `Watchdog` alert that fires permanently and is routed to something that pages when it stops arriving. The version I care about for telecom is narrower: per-signal liveness for the signals whose silence is dangerous.

```yaml
groups:
  - name: liveness
    rules:
      # Fires forever. Downstream pages when this stops arriving.
      - alert: Watchdog
        expr: vector(1)
        labels: { severity: none }

      # Silence on a saturated signal is not recovery.
      - alert: SaturatedSignalWentSilent
        expr: |
          (max_over_time(node_filesystem_used_pct{job="edge-media"}[6h]) > 90)
          unless
          present_over_time(node_filesystem_used_pct{job="edge-media"}[20m])
        for: 5m
        labels: { severity: page }
        annotations:
          summary: "{% raw %}{{ $labels.instance }}{% endraw %} was above 90% and stopped reporting"
          description: >-
            The exporter or the host is gone. Treat as full, not as recovered.
```

The `unless` is the whole trick. It encodes "this was dangerous, and now I cannot see it," which is the state a purely presence-based board structurally cannot express.

One detail worth stealing, because I got it wrong first: the obvious version of this uses `and absent_over_time(...)`, and it never fires. Binary operators match on the full label set, and `absent_over_time` only carries the labels you wrote into the matcher, so it has no `instance` to join against. `unless present_over_time(...)` keeps both sides on the same per-series labels and does what you meant.

## 4. Making the summary testable

The thing I should have built on day one, and built on day nine instead, is a reconciliation check. If a summarizer sits between the alerting system and the consumer, that summarizer is a piece of production software and it needs a test that fails when it drops something.

```python
def reconcile(source_fps: set[str], payload_fps: set[str], prev: set[str]) -> dict:
    """Compare what the alerting API says against what the summary emitted."""
    dropped = source_fps - payload_fps          # summarizer lost an active alert
    phantom = payload_fps - source_fps          # summarizer invented one
    closed  = prev - source_fps                 # real resolutions, event-derived
    return {"dropped": dropped, "phantom": phantom, "closed": closed}


if __name__ == "__main__":
    src  = {"a", "b", "c"}
    seen = {"a", "c", "z"}
    prev = {"a", "b", "c", "d"}
    r = reconcile(src, seen, prev)
    assert r["dropped"] == {"b"}, r
    assert r["phantom"] == {"z"}, r
    assert r["closed"]  == {"d"}, r
    print("ok")
```

Twenty lines. It would have caught the grouping collapse, both directions of the `resolved` bug, and the empty payload, because during the three-hour metrics outage `source_fps` was populated and `payload_fps` was not, which is a screaming `dropped` set rather than a peaceful `active: []`.

Note that `closed` is derived from set difference across polls, not from a field. That is the event-shaped resolution signal from section 2, and it is one operator.

## 5. What this says about the autonomous NOC

The industry conversation about agentic operations is almost entirely about the reasoning layer: correlation quality, root-cause suggestions, runbook execution, how much autonomy to grant. Having now run a reasoning layer over a real telecom alerting stack for two weeks, at fifteen-minute cadence, I think that emphasis is backwards.

The model side was, frankly, the easy part. Correlating a media-node crash with a websocket disconnect burst and a set of stuck call legs ninety seconds later is well within reach. So is noticing that a burst of alerts landing within twenty-four seconds of each other on the same cluster is one common-mode failure rather than five incidents.

What was not within reach was inferring the existence of an alert that never appeared in the input. No amount of reasoning recovers a dropped fingerprint. An autonomous NOC inherits, exactly and without appeal, the fidelity of the interface it reads, and today that interface is usually a chat-shaped summary designed for a human who will scroll up when something feels off. Humans have out-of-band recovery. They notice the room is too quiet. An agent reading a well-formed JSON payload containing `active: []` has no such instinct, and the payload is syntactically perfect.

So the ordering I would now argue for is: before you grant an agent any autonomy, give it a machine-readable alert interface with instance-level identity, event-shaped state transitions, an explicit liveness signal, and a reconciliation test that fails loudly. That is unglamorous plumbing. It is also the entire difference between an agent that triages and an agent that hallucinates calm.

## What to actually take away

**Your triage layer is a system under test, and its failure mode is silence.**

Concretely, four things. Track state per instance fingerprint, never per rendered notification, because grouping is lossy compression and the loss is exactly the thing you page on. Derive resolution from observed state transitions, never from a status field, because that field describes your parser's coverage rather than your infrastructure. Encode dangerous silence explicitly with `absent_over_time` guards on signals that were already saturated, because absence cannot be ranked in a list of present things. And reconcile the summary against the source every cycle, so a broken pipeline reports as a broken pipeline instead of as a quiet night.

Do that, and the reasoning layer, human or model, has something honest to reason about. Skip it, and every hour of quiet you see is unfalsifiable.
