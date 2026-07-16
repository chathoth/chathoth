# Systems That Lie: What Verifying Six Infrastructure Projects Taught Me

Fifteen years into platform engineering, the failure mode I keep
running into isn't "I didn't know the tool." It's simpler and harder
to catch: **I trusted a system's own report of itself instead of
checking what it actually did.** A status code, a log line, a model's
explanation of its own reasoning — all of these are claims, not facts.
The gap between "it said it worked" and "it worked" is where real
incidents live, and it's rarely obvious until you've gone looking for
it on purpose.

Over a series of projects — a security linter, three Terraform
modules, an AI agent lab, an observability exporter — I made verifying
that gap the actual discipline, not an afterthought. Every claim in
every one of these repos is backed by something I actually ran, not
something I assumed would happen. That discipline kept surfacing the
same shape of bug in completely unrelated technologies.

## 1. An API can return "success" while telling you it failed

Building [`marketo-observability-exporter`](https://github.com/chathoth/marketo-observability-exporter),
I confirmed something directly against Adobe's own developer docs and
then reproduced it in a real, running client: **Marketo's REST API
returns HTTP `200 OK` even when it's rejecting your request** — for
rate limiting, for an expired token, for exceeding your daily quota.
The real outcome lives one layer down, in the JSON body:

```json
{ "requestId": "e42b#14272d07d78", "success": false,
  "errors": [{ "code": "606", "message": "Max rate limit exceeded" }] }
```

A client that only checks the HTTP status — which is the natural,
default thing to check — would treat a throttled, rejected request as
if it had succeeded. I wrote a test that asserts both facts at once:
the raw status really is 200, and the client still correctly raises,
specifically because it checks the `success` field instead. It's a
small thing. It's also exactly the kind of small thing that turns
into "why did leads stop syncing three days ago and nobody noticed"
if you don't build for it.

## 2. An AI agent can narrate an action and then believe it happened

This is the one I find most unsettling, and it came out of
[`AI-Platform-Lab`](https://github.com/chathoth/AI-Platform-Lab)'s
guarded runbook agent. The agent was supposed to check disk usage,
then — only if a threshold was actually confirmed — clean up old logs.
The first fix (force the model to make one tool call at a time instead
of batching several before observing any result) closed one real
bug. But a second, subtler one showed up under testing: after a
multi-step runbook, the model would sometimes **write out, in plain
text, "I'll now call `cleanup_old_logs`..."** — and then, on the next
turn, behave as if that call had already happened, moving straight to
the next step without ever actually invoking the tool.

Nothing crashed. Nothing errored. The transcript read as if the
runbook had executed correctly. If anything downstream had trusted
that narration as confirmation — a log line, a status dashboard, a
human skimming the output — a required action would have silently
never run. The fix wasn't "trust the model more carefully"; it was
architectural: a destructive action only counts if it comes through
as a real, structured tool call, never as text, and the guardrail that
enforces that lives in code the model doesn't control. I later
reproduced the same underlying risk (batched tool calls, no
observation step in between) in LangChain's `bind_tools()` too — same
bug, different framework, which told me this isn't a quirk of one
library, it's a property of how these models behave by default.

## 3. A testing framework can fabricate data that looks real

Building the Terraform-based projects
([`terraform-aws-secure-baseline`](https://github.com/chathoth/terraform-aws-secure-baseline),
[`terraform-aws-landing-zone`](https://github.com/chathoth/terraform-aws-landing-zone),
[`terraform-elasticstack-lifecycle`](https://github.com/chathoth/terraform-elasticstack-lifecycle)),
I verified everything statically wherever real cloud credentials
weren't safe to use, via Terraform's native `mock_provider` testing.
It turns out the mock is a liar too, just a well-intentioned one:
`mock_provider` fabricates a plausible-looking fake value for *any*
computed attribute it can't determine — including for data sources
that make no real API call at all and are pure local computation.
The fake JSON string it invents for one of those isn't actually valid
JSON, which breaks a completely different resource's own schema
validation several layers downstream, producing an error that reads
like "your code is broken" when the actual problem is "the test
double lied about what it would return."

The fix each time was `override_data`/`override_resource` — supplying
a realistic value explicitly rather than trusting the framework's
default fabrication. The broader lesson: a testing tool's job is to
let you verify real behavior; it is not automatically trustworthy just
because it's the thing meant to catch bugs.

## The pattern, and what I actually do about it now

Four completely unrelated systems — a SaaS marketing platform, a
large language model, an infrastructure testing framework, and (from
an earlier project) a vector database whose `add()` call silently
no-ops on a duplicate ID instead of erroring — all shared the same
failure shape: **the system's self-report and its actual effect
quietly diverged, and nothing about the interface made that obvious.**

What I changed as a result, and now apply by default:

- **Check a second, independent signal, not just the first one
  offered.** An HTTP status is not proof of outcome; the response body
  usually is. A model's tool call list is proof of action; its prose
  is not.
- **Never let narrated intent count as executed action.** If a system
  claims it did something, the only thing that counts is the
  structured, auditable record of it actually happening — not a
  description of it.
- **Distinguish "the tool says this is valid" from "this actually
  works."** Static validation and live verification catch different
  classes of lies. I run both, deliberately, rather than treating a
  green checkmark from either one as sufficient on its own.
- **Document the gap when you find it, not just the fix.** Every repo
  linked above has a section describing the real bug that verification
  caught, not a cleaned-up version of events. The bug is usually more
  instructive than the feature.

## The portfolio, organized by what each one actually proves

- **[`dispatcher-lint`](https://github.com/chathoth/dispatcher-lint)** — catches configuration that *looks* complete but doesn't actually enforce what it claims to (verified live: an Apache directive that should have blocked the `TRACE` HTTP method didn't, because Apache serves `TRACE` from a path the directive never touched).
- **`terraform-aws-secure-baseline` / `terraform-aws-landing-zone` / `terraform-elasticstack-lifecycle`** — infrastructure-as-code where every claim about what a module enforces is backed by either a real deployment or an explicitly-overridden, honestly-labeled test double, never just "the plan looked right."
- **`AI-Platform-Lab`** — a hands-on account of where AI agents reliably fail in production-shaped scenarios, and the specific, verified guardrails that close each gap.
- **`marketo-observability-exporter`** — treats a SaaS platform you don't control as something to actively monitor rather than trust, precisely because its own API can't always be taken at face value.

Different technologies, same discipline underneath: assume nothing
reports its own state honestly until you've checked, and build the
thing that checks.
