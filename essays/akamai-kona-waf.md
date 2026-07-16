# Akamai Kona / App & API Protector: How the Anomaly-Score WAF Actually Works

*Reference notes on Akamai's WAF product — still widely called "Kona
Site Defender" (the legacy name), now sold as "App & API Protector."
These are general technical notes on how the product works and where
teams typically get tripped up tuning it, not a report of live testing
against a specific system.*

> **TL;DR** — Kona doesn't block on a single matched rule. It scores a
> request across attack categories and blocks when a cumulative score
> crosses a threshold — which changes almost everything about how you
> tune it, roll it out, and debug a false positive.

---

## 1. It's an anomaly score, not a switchboard of on/off rules

The Kona Rule Set (KRS) groups its rules by attack category — SQL
Injection, XSS, Local/Remote File Inclusion, Command Injection, HTTP
Policy, Protocol Violations, Web Attack Tool detection, and a few
others. Each rule that matches a request adds points to that
category's running score for the request. The WAF acts (alert or
deny) when a category's score crosses a configured threshold — not
the instant any single rule fires.

The practical consequence: disabling the one rule ID you see in a
false-positive alert often doesn't fix it. If three lower-weight rules
in the same category all matched the same request, removing one still
leaves the combined score over threshold. Tuning has to look at the
whole scored request, not the rule that happened to appear at the top
of the alert.

## 2. Alert mode is a rollout stage, not a permanent setting

Every new rule, every retuned threshold, should go live in **Alert**
mode first — logged, scored, visible in the Security Events report,
but not blocking. The mistake is tuning against a short or
unrepresentative traffic sample and then flipping straight to Deny.
Traffic shapes that don't show up in a quick sanity check — month-end
batch jobs, a mobile app's non-browser user agent, a partner's
server-to-server integration with an unusual content type — are
exactly the traffic that gets silently blocked days later if Alert
mode didn't run long enough to see it.

## 3. Scope exceptions to the match condition, not the rule group

When a legitimate payload trips a rule — a CMS rich-text field
containing `<script>` in sample/commented content, a JSON API body
with SQL-looking keywords in a free-text field — the fix is a
conditional exception scoped to the exact path, parameter, header, or
cookie that's legitimately triggering it. Disabling an entire attack
category on a whole path prefix because one endpoint under it has a
false positive removes protection for every other endpoint sharing
that prefix. The blast radius of the fix should match the blast
radius of the problem, and with a path-prefix-scoped exception it
usually doesn't.

## 4. Rate controls are a distributed estimate, not an exact counter

Akamai enforces rate-based rules using counters local to each edge
server, reconciled periodically rather than tracked against one
global atomic counter in real time. A policy written as "500 requests
per minute" is an approximation across however many edge machines see
the traffic — it converges quickly on concentrated traffic (an attack
hitting a small number of edge locations) and is much looser on
traffic that's naturally spread across Akamai's whole edge network.
Rate controls are better understood as a burst/concentration detector
than a precise global throughput cap, and policies should be designed
around that, not around an assumption of exact enforcement.

## 5. Managed rule-set updates can silently unpick your tuning

Akamai periodically updates the managed Kona Rule Set — new detection
logic, retuned scores, renumbered or split rule IDs. A property set
to auto-update gets new attack coverage for free, but a conditional
exception scoped to a specific rule ID can stop matching if that ID
changes shape in an update, silently reintroducing a false positive
you'd already fixed. Pinning to a fixed KRS version avoids that but
forfeits new coverage until you deliberately upgrade. Neither choice
removes the need to do this: **re-run the Alert-mode validation pass
after every rule-set change**, not just at initial rollout — treat a
managed rule-set update the same as any other change to a security
control in front of production traffic.

## 6. The WAF only protects traffic that actually reaches it

Kona evaluates requests that arrive through Akamai's edge. If the
origin's real IP is discoverable — DNS history, a certificate
transparency log entry for a direct-to-origin hostname, a
misconfigured health-check endpoint answering on the public internet
— and the origin doesn't restrict inbound connections to Akamai's
published IP ranges, an attacker can reach the origin directly and
the WAF never sees the request at all. The control that actually
makes the WAF meaningful isn't the WAF configuration itself; it's the
origin firewall rule enforcing "Akamai or nothing." A well-tuned
Kona policy in front of an origin that still accepts direct internet
traffic is protecting against nothing.

## 7. Bot Manager and App & API Protector evaluate independently

Bot mitigation and the WAF are separate products with separate
policies, and by default they don't share state. A request can be
classified as a legitimate, allow-listed bot by Bot Manager and still
get denied by Kona's rule evaluation, or the reverse. Troubleshooting
a blocked request that "shouldn't have been blocked" means checking
both the Bot Manager and the Security Events views — a clean result
in one doesn't tell you anything about the other.

## 8. Evaluation order matters for custom rules

Custom rules and the managed rule set don't necessarily evaluate in
the order you'd assume, and where a custom allow/bypass rule sits
relative to KRS evaluation determines whether it actually does what
it's meant to. A custom rule written to exempt a specific path from
WAF blocking only works if it's positioned and scoped so it's
evaluated ahead of the managed rules it's meant to override — placed
wrong, it's a no-op that looks correct in the configuration UI but
never actually fires first.

## 9. The dashboard is a summary, not the evidence

The Security Events report in Akamai's control center is built for
at-a-glance triage, and under high event volume it aggregates and
samples rather than showing every event. Real forensic work — proving
exactly which rule and score combination fired, on which request,
with which headers — needs the raw event export via SIEM/DataStream
integration, not the dashboard view. Treat the dashboard as the
index, and the raw event log as the source of truth.

---

Same underlying discipline as the rest of this stack: the control
plane's summary of what it did is a starting point for investigation,
not the final answer. What actually happened is in the raw event
data, one layer down from the dashboard that describes it.
