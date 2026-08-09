---
layout: post
title:  "Per-path SLOs need packet-level evidence, not site averages"
description: "Site averages compress away the one path that is actually broken. How eBPF packet-level evidence and OTLP let you define SLOs at the granularity your service really fails at, without blowing up your time-series database."
date:   2026-08-07 09:00:00 -0300
categories: Concepts
permalink: /:categories/:title
---

Every carrier-grade postmortem I have read in the last few years has the same shape. A voice or video service degrades for a subset of subscribers. The dashboards are green: the site is up, the node is up, CPU is fine, the availability SLO for the month still reads 99.98%. Somewhere in the middle of the network, one path out of dozens was dropping 0.4% of packets and adding 12 ms of jitter, and the monitoring stack averaged it into invisibility.

That is not a tooling gap in the "we need a better dashboard" sense. It is a modeling gap. We keep defining SLOs at the granularity that is cheap to measure (per site, per node, per cluster) instead of the granularity at which the service actually fails, which is *per path*. And until recently, per-path measurement in a cloud-native core was genuinely hard: you either bolted on network probes that knew nothing about your workloads, or you settled for flow-level statistics that told you *something bad happened around here* without ever letting you follow one packet.

Two things changed in 2026 that make per-path SLOs practical instead of aspirational: eBPF data planes got packet-level trace identity, and OTLP became a realistic single spine for telco telemetry. This post is about wiring those two together without setting your time-series database on fire.

## 1. Site averages are a lie your TSDB tells you cheaply

Start with the arithmetic, because it is the whole argument.

Suppose a site aggregates 40 paths. Thirty-nine of them have 1 ms jitter, one has 40 ms. The site average is 1.97 ms, comfortably inside any sane budget. Meanwhile the subscribers on that fortieth path are experiencing broken VoLTE calls, choppy video, and RTP streams that the jitter buffer cannot rescue.

Averaging is the cheapest possible compression, and it compresses away exactly the signal you care about. Percentiles help, but a p99 over 40 paths still hides a single sustained offender if it does not carry enough traffic weight. The only representation that survives contact with reality is one that keeps the path as a dimension.

The counter-argument is always cost, and it is a real one. Path is a high-cardinality label: source pool × destination pool × network function × direction grows fast, and if you naively add `src_ip`/`dst_ip` you have invented an unbounded label set and a Prometheus outage. The answer is not to give up on per-path measurement. The answer is to be deliberate about what "path" means.

A path, for SLO purposes, is a **bounded equivalence class of routes that share fate**: an ingress site to an egress site, over a given transport class, for a given traffic type. Something like:

```
path="edge-sp01 -> core-sp02"
transport="backhaul-primary"
class="voice-rtp"
```

That is three labels with enumerable values. If you have 20 sites and 3 transport classes and 4 traffic classes, your worst case is a few hundred series per metric, which is entirely affordable. What you must *not* do is let per-subscriber or per-5-tuple identity leak into label space. Keep those in traces and logs, where high cardinality is priced correctly, and keep metrics on the bounded path key.

## 2. eBPF gives you the packet, not just the flow

Flow-level telemetry answers "how many bytes moved between A and B in this window." That is a statistical view. It can tell you loss happened; it cannot tell you *which* packet, *where* it was dropped, or whether reordering or a retransmit storm caused what looked like loss.

The eBPF data planes have been closing this gap steadily, and the recent addition of per-packet trace identity in [Hubble-style flow observability](https://docs.cilium.io/en/stable/observability/hubble/) is the interesting step. Being able to filter observed flows by an IP packet trace ID means you can follow a single packet through the eBPF programs at TC ingress, the conntrack lookup, the policy verdict, the egress hook, and see exactly where it stopped. Micro-loss debugging moves from statistical inference to deterministic evidence.

For anyone monitoring containerized network functions this matters more than it does for a typical web microservice fleet. A CNF's failure modes are packet-shaped: reordering that breaks a signaling state machine, a 3 ms scheduling stall that pushes an RTP frame past its playout deadline, a policy drop that only fires on one direction of a bidirectional session. None of those look like anything on a CPU graph.

A minimal, vendor-neutral shape of this — a tracepoint-attached program that classifies drops by path and reason:

```c
struct drop_key {
    __u32 path_id;   // hashed from bounded path enum, NOT raw IPs
    __u32 reason;    // kernel drop reason code
};

struct {
    __uint(type, BPF_MAP_TYPE_PERCPU_HASH);
    __type(key, struct drop_key);
    __type(value, __u64);
    __uint(max_entries, 4096);
} drops SEC(".maps");

SEC("tracepoint/skb/kfree_skb")
int on_kfree_skb(struct trace_event_raw_kfree_skb *ctx)
{
    struct drop_key k = {};
    k.path_id = path_id_from_skb(ctx);   // resolves to your bounded path set
    k.reason  = ctx->reason;

    __u64 *cnt = bpf_map_lookup_elem(&drops, &k);
    if (cnt)
        __sync_fetch_and_add(cnt, 1);
    else {
        __u64 one = 1;
        bpf_map_update_elem(&drops, &k, &one, BPF_ANY);
    }
    return 0;
}
```

Two design notes worth stealing. First, `path_id` is resolved *in kernel* against a bounded map, so cardinality is capped by construction rather than by hoping the exporter behaves. Second, keeping the kernel drop `reason` turns "we lost packets" into "we lost packets because the policy verdict denied them," which is the difference between a two-hour and a ten-minute incident.

## 3. OTLP as the spine, so you stop running three pipelines

The second half of the problem is transport. The historical telco pattern is three parallel telemetry pipelines: a vendor NMS for network element KPIs, a Prometheus stack for the Kubernetes layer, and application logs going somewhere else entirely. Correlating across them is a manual, tribal-knowledge exercise performed at 3 a.m.

The pragmatic move is to treat [OTLP](https://opentelemetry.io/docs/concepts/what-is-opentelemetry/) as the common wire format and the Collector as the place where policy lives — enrichment, redaction, sampling, routing — rather than duplicating that logic in every exporter. This is less of a bet than it was a couple of years ago: OTLP receivers and exporters now ship in the tooling on both sides of the fence, and the telco vendors themselves have started describing OTLP as the collection and forwarding path for network function telemetry rather than as an application-only concern.

Concretely, that means your eBPF exporter emits OTLP metrics with the same `path`, `transport`, and `class` attributes that your CNF application spans carry. Then a jitter spike and a signaling retry burst are joinable on a shared key instead of on a human's memory of which site maps to which cluster.

The Collector is also where you enforce the cardinality contract:

```yaml
processors:
  transform/path_only:
    metric_statements:
      - context: datapoint
        statements:
          - delete_key(attributes, "src_ip")
          - delete_key(attributes, "dst_ip")
          - delete_key(attributes, "session_id")

  filter/known_paths:
    metrics:
      datapoint:
        - 'attributes["path"] == nil'

exporters:
  prometheusremotewrite:
    endpoint: http://tsdb.internal:9090/api/v1/write
```

Drop the unbounded identifiers before they reach the TSDB, and drop datapoints that failed path resolution instead of letting them through as an `unknown` bucket that quietly becomes your largest series. If a lot of traffic lands in "unresolved," that itself is an alert — your topology model has drifted from reality.

## 4. Writing the SLO so it means something

Once path is a first-class dimension, the SLO writes itself. For a voice path, the thing that actually breaks the service is jitter exceeding the playout budget for a sustained window, not availability.

{% raw %}
```yaml
groups:
  - name: path-slo
    interval: 30s
    rules:
      - record: path:slo_good_ratio:5m
        expr: |
          sum by (path) (rate(network_path_jitter_ms_bucket{le="20", class="voice-rtp"}[5m]))
          /
          sum by (path) (rate(network_path_jitter_ms_count{class="voice-rtp"}[5m]))

      - alert: PathJitterBudgetBurn
        expr: |
          (1 - path:slo_good_ratio:5m) > (14.4 * 0.001)
        for: 10m
        labels:
          severity: page
        annotations:
          summary: "Fast burn of jitter budget on {{ $labels.path }}"
          description: >-
            This path is burning its 99.9% jitter budget 14.4x faster than
            sustainable: over 10 minutes, more than 1.44% of voice datapoints
            exceeded the 20 ms budget. Site-level availability may still be green.
```
{% endraw %}

Note what this does *not* do. It does not alert on jitter crossing a threshold once that pages you for every transient. It alerts on **burning the error budget** for a specific path at a rate that would exhaust the month, which is the only alert condition an on-call engineer can act on without resentment. The 14.4x multiplier is not arbitrary either — it is the standard fast-burn factor from [the SRE workbook's chapter on alerting on SLOs](https://sre.google/workbook/alerting-on-slos/), the rate at which you would consume a 30-day budget in about two days. And because the label set is bounded, `sum by (path)` stays cheap.

One implementation detail that will bite you on the way there: `le="20"` is an exact string match against a bucket boundary, so 20 ms has to actually *be* one of your configured [histogram buckets](https://prometheus.io/docs/practices/histograms/). If it is not, the numerator silently returns nothing and the ratio reads as a permanent zero, which looks exactly like a totally broken path. Pick your buckets from the budgets you intend to alert on, not from a default set.

The other half of making this real is training the SLO before you commit to it. Run the recording rules in shadow mode for two or three weeks, look at the actual distribution per path, and set the budget from observed behavior plus the physical constraint (a 20 ms playout budget is a codec fact, not a negotiation). An SLO derived from a vendor datasheet will either page constantly or never fire.

## What to actually take away

The lesson is not "adopt eBPF" or "adopt OpenTelemetry." It is that **your SLO granularity should match your failure granularity, and everything else is an implementation detail in service of that.**

If your services fail per path, measure per path. Use a bounded path key so the cost stays sane. Use eBPF where you need deterministic packet-level evidence rather than statistical flow summaries. Use OTLP so that evidence lands in the same attribute space as your application traces. Then write burn-rate alerts against a budget you derived from observation, not from a datasheet.

The uncomfortable part is that doing this correctly usually reveals your availability numbers were flattering you. That is the point. A monitoring stack that only ever confirms things are fine is not a monitoring stack, it is a compliance artifact.
