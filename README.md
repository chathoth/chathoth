# Hi 👋, I'm Irfad Chathoth

**Platform Engineer | DevOps | SRE | AI | Cloud** — 15+ years building
enterprise platforms. The thread running through everything below:
verify what a system actually does, not what it reports about itself
— see [Systems That Lie](./essays/systems-that-lie.md).

## 🔭 Currently exploring

AI Agents · RAG · Local LLMs (Ollama) · Terraform Automation · AWS ·
Elasticsearch

## 🧰 Tech Stack

![AWS](https://img.shields.io/badge/AWS-232F3E?style=flat&logo=amazonaws&logoColor=white)
![Azure](https://img.shields.io/badge/Azure-0078D4?style=flat&logo=microsoftazure&logoColor=white)
![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=flat&logo=terraform&logoColor=white)
![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=flat&logo=ansible&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=flat&logo=jenkins&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=flat&logo=linux&logoColor=black)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![Elasticsearch](https://img.shields.io/badge/Elasticsearch-005571?style=flat&logo=elasticsearch&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=flat&logo=grafana&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=flat&logo=prometheus&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=flat&logo=githubactions&logoColor=white)

## 📦 Portfolio

Six projects, organized by what each one actually proves - not just a
repo list. Every claim in every one of them is backed by something
that was actually run, not assumed.

| Repo | What it proves |
|---|---|
| [AI-Platform-Lab](https://github.com/chathoth/AI-Platform-Lab) | Hands-on AI/LLM fundamentals through agents and RAG, written from a platform engineering perspective - plus two end-to-end projects (incident search, a guarded runbook agent) |
| [dispatcher-lint](https://github.com/chathoth/dispatcher-lint) | A security linter that catches AEM Dispatcher/Apache config that *looks* complete but doesn't actually enforce what it claims to - verified live against a real container |
| [terraform-aws-secure-baseline](https://github.com/chathoth/terraform-aws-secure-baseline) | An account-level AWS security baseline module, verified statically with real overrides, not assumed mock defaults |
| [terraform-aws-landing-zone](https://github.com/chathoth/terraform-aws-landing-zone) | Org-level AWS governance (Organizations, SCPs, Control Tower, Identity Center), same verification discipline |
| [terraform-elasticstack-lifecycle](https://github.com/chathoth/terraform-elasticstack-lifecycle) | Elasticsearch ILM/SLM lifecycle policies, verified with a real local cluster and a real triggered snapshot - not mocked |
| [marketo-observability-exporter](https://github.com/chathoth/marketo-observability-exporter) | Prometheus/Grafana observability for a SaaS platform you don't control, built read-only by design |

## ✍️ Writing

### [Systems That Lie](./essays/systems-that-lie.md)

The pattern I keep finding across completely unrelated technologies:
a system's report of its own state and what it actually did quietly
diverge, and nothing about the interface makes that obvious. Marketo
returns HTTP `200 OK` on a rejected, rate-limited request. An AI agent
narrated a fix in plain text and then behaved as if it had actually
run it. Terraform's own test-mocking framework fabricates plausible
fake data that breaks downstream validation. Same shape of bug, three
unrelated systems — and what I now check for by default because of it.

### [CDN Cache Forensics: Failure Modes Nobody Documents](./essays/cdn-cache-forensics.md)

Field notes on a multi-tier caching stack where "caching is enabled"
and the CDN bill kept growing anyway: frozen header snapshots that
outlive a fix, revalidation loops that preserve the bug they should
heal, TTLs that silently count down to zero, and the header-level
forensic techniques that catch each one.

### [Akamai Kona / App & API Protector: How the Anomaly-Score WAF Actually Works](./essays/akamai-kona-waf.md)

Reference notes on the anomaly-scoring model underneath Akamai's WAF —
why disabling one flagged rule often doesn't fix a false positive, why
rate controls are a distributed estimate rather than an exact counter,
and why a well-tuned policy protects nothing if the origin still
accepts direct traffic.

## 📫 Reach me

Toronto, ON · Publicis Sapient · [LinkedIn](https://www.linkedin.com/in/irfad-chathoth-42319a1bb)
