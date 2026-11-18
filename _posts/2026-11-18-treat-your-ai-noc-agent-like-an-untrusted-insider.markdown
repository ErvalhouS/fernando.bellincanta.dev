---
layout: post
title:  "Treat your AI NOC agent like an untrusted insider"
description: "An incident-response agent on staging, and the four controls outside the model it needed before I would consider it for production."
date:   2026-11-18 09:00:00 -0300
categories: Concepts
permalink: /:categories/:title
---

The incident agent we started from would have told anyone in our Slack what a colleague had said to it in a private message. Nobody would have needed an exploit, only a question.

That came from a search tool the agent framework ships by default, and I found it reading the framework's source before we deployed. It set the rule for everything after: anyone who can put text in front of the agent can make it misuse whatever it holds.

An AI NOC agent reads the same alerts, logs, Slack threads and GitHub issues I do, and plenty of that text was written by people I have never met. Any of it can carry an instruction, and the model has no reliable way to tell an instruction in a log line from one in my message. It sits inside with real access and takes orders from outside, which is about as close to an insider threat as software gets.

Here are the four controls ours needed before I would consider it for production, and where each one came from.

## 1. Search is a data leak before it is a feature

The session-search tool lets the agent check how a similar question was handled before. Useful, and wrong for us: it searched every session, including other people's direct messages with the bot.

Scoping the search per user would have meant trusting a filter in code we did not write, for a feature we barely needed. We removed the tool. **When a capability is worth less than its blast radius, deleting it is the one control that needs no maintenance.**

## 2. A denylist of credentials protects nothing else

The file-read tool came with a denylist of credential stores. It caught the obvious tokens and left everything else open. The agent could read its own state database, which holds every conversation from every user, and its own logs. A reply could attach any local file that was not on the denylist straight into Slack.

The replacement is an allowlist that fails closed, enforced by a hook that runs before every read. The check itself is two lines:

```python
def readable(path: str, cwd: str) -> bool:
    real = os.path.realpath(os.path.join(cwd or "/", os.path.expanduser(path or ".")))
    return any(real == root or real.startswith(root + "/") for root in READ_ROOTS)
```

`READ_ROOTS` holds the repo checkout and the agent's own inputs, and `realpath` resolves symlinks and `..` before the comparison, so a link planted inside the repo can't point back out.

Two more settings close the other doors. Media delivery is strict, so a reply cannot attach local files at all. And link previews are off:

```yaml
unfurl_links: false
unfurl_media: false
```

Slack's servers fetch every link the bot posts to build a preview, so an injected link with data in its query string would leak that data without anyone clicking it.

An allowlist only has to name the few paths the agent actually uses. That list is far easier to get right than a list of every dangerous one.

## 3. Assume the input is hostile

The agent's instructions list what it must never do. No control depends on it obeying them. Alert annotations, log lines and Slack messages all reach the model, and so does GitHub, where anyone can open an issue on a public repo. **You cannot defend a process boundary with prose.**

So the design assumes the injection succeeds. Nothing the agent can call changes state except three paths: a reply in Slack, a pull request that a person has to merge, and PagerDuty writes on an allowlisted service, where creating or reassigning an incident waits for a person's yes in the thread. The GitHub tools filter text by who wrote it rather than by which repo it sits in: issue and comment bodies on public repos come through only from org members, and logs from fork pull requests never do.

What an injected instruction can still produce is mostly bad advice, and a person reads that before acting on it. That check earns its keep even without an attacker. The agent's first pull request arrived about a minute after a crash-looping alert, with a confident diagnosis that was wrong: the health check settings it blamed were already in place. Someone read it and corrected it before it merged.

## 4. Keep the keys out of the agent's hands

Out of the box the containers ran as root, the agent process held the credentials for every service it talked to, and the only record of what it did was a log file inside its own pod. The agent could also reach its own configuration. On its first day on staging, the framework's self-improvement loop tried to edit the agent's playbook, and only the read-only mount stopped it. The file that lists who may talk to the bot in Slack was writable too, so in principle the agent could have widened its own audience.

Now writes are blocked by a hook before they run and turned into pull requests, scoped per agent. Tokens for PagerDuty, GitHub and Grafana live in other containers, and the model only reaches their fixed tools. The Slack and model keys the framework needs stay in its environment, which the agent can't reach: it has no shell, and the read hook blocks `/proc`.

Paging someone goes through a sidecar too. The agent can propose a PagerDuty incident, but the sidecar reads the confirmation from the Slack thread itself and checks that the yes came from the person who asked. **The model never gets to say who approved it.**

The pods got the same treatment:
- sidecars run non-root, with no capabilities and a read-only root filesystem
- the main container drops to an unprivileged user after a short root init
- no pod carries a Kubernetes service account token

An audit sidecar streams every tool call, block and error to Loki, so the trail outlives the pod. If the agent is ever wrong, someone can read exactly what it did, cut its access, and roll back to the last reviewed instructions.

## Ask for the threat model

Each change was the smallest one that closed the hole in front of me. Together they fit in one threat model where every control names the file that enforces it, next to the list of what is still open. Ours runs on staging only, and production waits on those open items.

That document is what I would ask for before trusting anyone else's agent. If a vendor can't show you which file stops their agent from handing one user's messages to another, assume nothing does. A week of setup costs less than the first private message your agent gives to the wrong person.
