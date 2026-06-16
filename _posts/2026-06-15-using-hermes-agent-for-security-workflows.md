---
layout: default
title: Using Hermes Agent for Security Workflows
Date: 2026-06-15 00:00:00 -0700
published: yes
tags:
    - hermes-agent
    - security
    - ai-agents
    - soc
    - appsec
    - hermes-as-a-security-agent-series
---

# Using Hermes Agent for Security Workflows

I have been using Hermes Agent for security work — triage pipelines, code-review gates, threat-intel summarization, compliance collection, incident response data analysis. This post is about where it fits, what it can do, and where it breaks.

<!--more-->

The National Vulnerability Database had roughly twenty-nine thousand CVEs backlogged as "Not Scheduled" — Harold Booth at NIST acknowledged at VulnCon26 that the pipeline had formally collapsed. Mandiant's M-Trends 2026 report added another dimension: exploitation now begins a mean of seven days before patches are available, and GPT-4 can autonomously exploit eighty-seven percent of one-day vulnerabilities given only the CVE description [15](#src-15).

This is what happens when machine-speed discovery meets human-scale triage with no guardrails. The offensive side has already learned that lesson the hard way. The defensive side is still figuring it out.

The numbers on defense are just as sobering. The ISC2 2024 Cybersecurity Workforce Study estimates **4.76 million unfilled security roles worldwide** [1](#src-1). SOCs process an average of **4,484 alerts per day** [2](#src-2), and seventy-one percent of analysts report burnout [20](#src-20). AI-assisted developers ship code three to four times faster, often without adequate AppSec controls [3](#src-3), [4](#src-4); the resulting security findings surge has been measured at **10× in some enterprises** [4](#src-4).

The work is real. The humans are not there. The natural temptation is to throw an AI agent at the problem and hope it absorbs the overflow. Without validation gates, profile separation, and human-in-the-loop checkpoints, that ends badly.

This post is about doing it the other way. I have been running **Hermes Agent**, an open-source autonomous agent framework built by Nous Research, for security workflows. It is not a coding copilot tethered to an IDE, nor a chatbot wrapper around a single API. It is a platform-agnostic agent designed to run continuously on a five-dollar VPS, a GPU cluster, or serverless infrastructure, reachable from twenty-plus messaging platforms [7](#src-7). When configured correctly, it triages alerts, reviews code, summarizes threat intelligence, and collects compliance evidence twenty-four hours a day. When configured poorly, it becomes another machine-speed noise generator.

I ran a pilot. The task was design review and code review for a security-sensitive codebase — 1,610 files, 168,233 lines of code, 349 binary files, spanning 13 languages — primarily an enterprise Java web application with tests in JUnit 4. The baseline was a roughly two-week turnaround from submission to sign-off, bottlenecked on a small team of senior engineers. We configured Hermes with a `security-reviewer` profile — read-only repo access, no merge permissions, sandboxed terminal, deterministic model settings. On every pull request it ran a checklist: secrets scan, dependency audit, OWASP Top 10 pattern match, and reachability correlation against known CVEs. Results posted to Slack with green, yellow, or red per category.

The two-week process collapsed to a few hours. The agent flagged 150 high and critical vulnerabilities. Five were false positives — a human reviewer dismissed them in seconds. The remaining 145 were real. Sixty of those had already been caught by the static analysis tools in the pipeline. The other eighty-five were net-new — things the tools and the manual review had both missed.

Those false-positive dismissals mattered because the Kanban task was blocked at triage, waiting for a person to verify before anything moved forward. The gate was simple: a Kanban column that blocked forward progress until a human clicked approve. No confidence threshold, no auto-escalate — just a rule that said nothing moves without a person looking at it. The gate held.

That pilot is the reason this post exists. The architecture works. The configuration discipline is the difference between a few hours of focused review and a noise generator that nobody trusts. What follows is what I learned about getting it right.

## What Hermes Is

The core differentiators matter for security work because they dictate what the agent can learn, remember, and break.

**Self-improving through skills.** After solving a complex problem, Hermes persists the procedure as a "skill" that loads into future sessions. Over time the agent gets faster at your specific workflows. A peer-reviewed benchmark measured a **+40% repeat-task speedup** after two weeks of use, with skill retrieval latency under ten milliseconds over a corpus of ten thousand skills [5](#src-5). The caveat: that benchmark measured general coding tasks, not live SOC workloads.

**Persistent memory across sessions.** Hermes stores cross-session memory via pluggable backends (built-in SQLite, Honcho, Mem0). It builds a model of your environment, preferences, and workflows, which means you spend less time repeating setup context [6](#src-6).

**Provider-agnostic.** Three hundred-plus models across twenty-plus providers (OpenRouter, Anthropic, OpenAI, DeepSeek, local models) with zero lock-in. You can swap providers mid-workflow [6](#src-6). This matters for security because it lets you pin a deterministic model version for audit reproducibility without marrying a single vendor.

**Sandboxed terminal backends.** Hermes supports six execution backends: `local`, `docker`, `ssh`, `modal`, `daytona`, and `singularity` [6](#src-6). For security work, `docker` or `modal` are the right defaults; they isolate untrusted code, intel parsing, and SAST execution from the host.

**Profiles.** You can run multiple isolated Hermes instances with separate configs, sessions, skills, and memory [6](#src-6). This is non-negotiable for security: the profile that reads threat intel must not share tool access with the profile that posts to Slack or writes to S3.

The system's center of gravity is a conversation loop that calls an LLM with tool schemas, dispatches tool calls, and appends results. Surrounding that core are a SQLite session store, a tool registry, a messaging gateway, a cron scheduler, a Kanban work queue, and a delegation system for subagents [6](#src-6). Part 2 of this series covers what I set in each config section and why.

## Where It Fits in a Security Program

Hermes maps onto five functions that burn the most human hours. Four are drawn from active pilots; one is still in the works. Each includes an example and a hard constraint.

Three of these — threat intel summarization, incident response data analysis, and compliance evidence collection — draw from a single pilot that centralizes operational and threat data into a queryable surface. Code review runs independently on a separate profile and codebase. Vulnerability triage is still in the works.

### Vulnerability Triage
**What:** Enrich incoming alerts with CVE data, asset context, and reachability analysis before a human ever opens the ticket.

**Example:** This one is still in the works — the pilot isn't running yet. The plan: when a SAST tool flags a dependency, Hermes queries the NVD (or EUVD), checks whether the vulnerable function is reachable in the codebase, and drafts a triage note with CVSS, EPSS, and patch availability. If confidence is low, it blocks the task for human review. The architecture is straightforward — read-only, sandboxed, kanban-gated — but the integration with the SAST pipeline needs more work before I can start measuring results.

**Constraint:** GPT-class models exploit CVEs at eighty-seven percent success from descriptions alone [21](#src-21). Triage agents must be **read-only**; never grant them production system access.

### Code Review / SAST Augmentation
**What:** Gate pull requests on security checklists, correlate SAST findings with reachability, and draft remediation patches.

**Example:** On every PR to `main`, a cron job spawns a Hermes subagent with a `security-reviewer` profile. The agent runs a checklist (secrets scan, dependency audit, OWASP Top 10 pattern match) and posts a summary to Slack with green/yellow/red status per category. In the pilot described above, this flagged 150 high and critical vulnerabilities — 85 of them net-new findings that both the static analysis tools and manual review had missed.

**Constraint:** Seventy-two percent of enterprises use AI for code generation, but eighty-three percent ship without adequate AppSec controls [3](#src-3). In the pilot, the agent caught eighty-five net-new high and critical vulnerabilities that both the static analysis tools and manual review had missed.

### Threat Intel Summarization
**What:** Correlate threat intel feeds with operational data so engineers can ask "is this relevant to us?" and get an answer in seconds instead of hours.

**Example:** The same pilot that centralizes operational data also ingests threat intel — CISA KEV, vendor advisories, and the EUVD. An engineer can ask "does this CVE affect any of our running services?" and the agent cross-references the intel against the environment data already collected. The morning briefing use case — summarization, IOC extraction, prioritization — is the natural next step, but for now the value is correlation: connecting outside threats to inside reality without a human having to join two spreadsheets.

**Constraint:** Treat every fetched document as a **prompt injection surface**. CVE-2025-32711 (EchoLeak, CVSS 9.3) demonstrated that a single crafted email could cause M365 Copilot to silently exfiltrate internal files and data across Microsoft 365 services with zero user interaction [9](#src-9) CVSS score via NVD. Unit 42 documents **twenty-two active evasion techniques** in the wild, including zero-font text, off-screen positioning, Unicode zero-width characters, and Base64 encoding [14](#src-14).

### Incident Response Data Analysis
**What:** Give engineers as much context as possible during an active incident, without them having to hunt across a dozen dashboards.

**Example:** We have a pilot running that centralizes data from across the environment — auth logs, deployment events, config changes, alert history — into a queryable surface that an AI agent can reason over. During an incident, an engineer asks "what changed in the last four hours that could explain this spike in auth failures?" The agent correlates across systems and surfaces connections a human would take twenty minutes to assemble manually. It stops at insight delivery — no containment actions, no automated response.

**Constraint:** Early results are promising — the agent is surfacing insights and data relationships faster than a human could make the connections. But this is still a read-only data layer, not a playbook executor. SOAR black-box automation produces lower-quality reports than manual analysis [2](#src-2). Before any automated containment, human-in-the-loop checkpoints are mandatory. The long-term path is toward playbook execution with kanban gates, but for now the scope is deliberately narrow: gather, correlate, surface. Let the human decide what to do with it.

### Compliance Evidence Collection
**What:** Gather audit evidence from across the environment and map it to control frameworks — without three engineers spending a week scouring systems.

**Example:** I have been using Hermes to pull compliance evidence from the same unified data source that feeds the IR and threat intel pilots, plus direct queries to IAM, access review logs, and encryption-at-rest configs. It tags each artifact with the relevant control ID — `NIST-800-53-SC-28`, `SOC2-CC6.1`, whatever your framework demands — and writes them to an append-only store. What used to take three engineers a full cycle now takes one person roughly two hours. The evidence is easier to gather, the tagging is consistent, and the audit trail is append-only by default.

**Constraint:** This is still the **lowest-risk, highest-value** use case. Read-only collection with append-only storage reduces tampering risk to near zero. If you are new to agentic security, start here — it teaches the configuration discipline without putting production systems in the blast radius.

## Pitfalls to Avoid

The failure modes are the same everywhere. What breaks a triage queue also breaks a SOC.

### Prompt Injection
Any file an agent reads is an attack surface. CVE-2025-32711 showed that a single crafted email can cause a Copilot agent to silently exfiltrate data [9](#src-9). AI-generated security findings now routinely include hallucinated endpoints, wrong parameters, and cargo-culted headers. GreyNoise Labs documented AI-generated PoCs targeting Cisco ISE that used completely wrong API paths [16](#src-16).

OWASP's guidance is blunt: no foolproof fix exists [8](#src-8). Run intel parsing in sandboxed terminals (`docker` or `modal`). Strip active content from documents before ingestion. Monitor for anomalous outbound connections.

### Automation Bias
**Forty-seven percent of security analysts** accept AI alerts without verification; thirty-seven percent exhibit confirmation bias [13](#src-13) [2](#src-2). When the noise floor rises, humans stop listening.

Measure analyst override rates. Sustained low rates signal dangerous complacency. Rotate analysts between agent-assisted and manual shifts to preserve skill currency.

### RAG Corpus Poisoning
Injecting **five malicious texts** into a knowledge database with millions of texts achieves roughly ninety percent targeted attack success [22](#src-22). AGENTPOISON achieves greater than eighty percent attack success rate via constrained optimization on agent memory [25](#src-25).

Sign and version-pin RAG corpora. Monitor embedding-space outliers. Isolate intel ingestion from execution profiles.

### Scope Creep / Excessive Agency
**Eighty-two percent of enterprises** have discovered unknown AI agents in the past year; only **twenty-one percent** have formal decommissioning processes [26](#src-26). The Morris II worm demonstrated that self-propagating agents require no code exploit [12](#src-12); as one survey put it, in the agentic supply chain, context is code [11](#src-11).

Agents accumulate permissions organically. A profile that started with read-only repo access gains a web-search tool during one task, a file-write tool during another, and a Slack-post tool during a third. Without quarterly permission reviews, the original scope is unrecognizable within months. Maintain an agent inventory. Enforce permission reviews quarterly. Use `kanban_block` on ambiguity — never let the agent infer scope from context.

### Lack of Reproducibility
The negative time-to-exploit metric from Mandiant is possible partly because defenders cannot reproduce patches fast enough to verify them. Your agent must not add to that problem. LLM outputs are stochastic. Two identical queries return different findings. For audit reproducibility, log model version, prompt templates, and decoding settings — and pin them to deterministic values when findings need to be regenerated [10](#src-10). Float a model version mid-audit and you cannot reproduce the result.

### Secrets in Durable State
Task bodies, summaries, and metadata in Kanban rows are **durable forever**. Credentials placed there persist in SQLite long after the task is done. This is the agent equivalent of committing secrets to git.

Never put credentials in task bodies or metadata. Use environment secrets or vault-injected tokens at runtime. Rotate any credential that has appeared in agent context.

Before the lesson, one practical note on where to get vulnerability intel now that the NVD pipeline has collapsed.

## Where Else to Get Vulnerability Intel

The NVD pipeline collapsed, and it is not coming back on its own schedule. Roughly twenty-nine thousand CVEs are backlogged as "Not Scheduled," and NIST only enriches entries in CISA KEV, federal software, or EO 14028 critical software. If your triage agent depends on the NVD as ground truth, point at least one of its intel sources somewhere else.

The most credible new public option is the **European Vulnerability Database (EUVD)**, run by ENISA under the NIS2 Directive and live since May 13, 2025 [23](#src-23). ENISA became a CVE Numbering Authority in January 2024. The database ships three dashboards: critical vulnerabilities, exploited vulnerabilities (a mirror of CISA KEV, narrower in scope), and EU-coordinated vulnerabilities (those handled by European CSIRTs). As of May 2025 it held about 264,920 records against NVD's 294,484 [24](#src-24). It is built on the open-source Vulnerability-Lookup project, so anyone can self-host the same correlation engine.

EUVD is useful but not a full NVD replacement yet. VulnCheck's May 2025 review still labeled it "beta," flagged slow API download times, modified EPSS scores that do not match the upstream EPSS scale, and missing CWE, CPE, and reference-tag metadata. If you wire it into a triage agent, treat the EPSS values as ENISA-specific and do not mix them with FIRST EPSS in the same score column. The related Cyber Resilience Act Single Reporting Platform becomes mandatory for manufacturers by September 2026 and is a separate system, but it is the next surface to watch for actively-exploited disclosures tied to EU-market products.

For now the practical posture is: NVD as the spine, CISA KEV as the exploited signal, EUVD as the EU-coordinated supplement, and the relevant national CSIRTs (CERT.PL, NCSC-FI, INCIBE, BSI) for jurisdiction-specific advisories. Your agent's triage config should pin which sources are authoritative and which are advisory, and your audit trail should record which source produced each enrichment.

## The Lesson

Anthropic's Mythos was tested by Daniel Stenberg on a hundred seventy-eight thousand lines of cURL code. It flagged five "confirmed" vulnerabilities. Four were false positives or had no security impact. One was low-severity. Stenberg's assessment: "I see no evidence that this setup finds issues to any particular higher or more advanced degree than the other tools have done before Mythos" [19](#src-19).

Mythos was a model pointed at a codebase, not a platform — no sandboxed execution, no human review gates, no audit trail. It was AI without guardrails. That is the ceiling: plausible, well-formatted, confidently wrong. Drata's Kristina Vevia calls this the shift from "obviously wrong" to "plausibly wrong," and it is the real danger. It means your review process has to improve at the same rate as the models, or faster [17](#src-17).

Hermes Agent is designed to make that improvement systematic. Skills and memory let the agent learn your environment rather than hallucinate it. Profiles and sandboxed terminals enforce separation between read-only research and write-capable operations. Kanban and cron provide durable, auditable, restart-safe execution. Deterministic model settings make findings reproducible. HITL gates force human judgment before irreversible actions.

The NVD pipeline did not fail because AI found too many bugs. Neither did cURL's bug-bounty program. They failed because the triage pipeline could not distinguish signal from noise at machine speed. The same asymmetry is coming for every SOC and AppSec team: attackers and noise generators operate at machine speed, while defenders remain stuck at human speed [18](#src-18).

A well-configured agent does not close that gap by replacing humans. It closes the gap by absorbing the noise — the repetitive triage, the false-positive SAST alerts, the compliance checkboxing — so that human analysts can focus on judgment, creativity, and adversarial thinking. That only works if you treat configuration discipline as a security control, not as bureaucracy.

Start with read-only tasks. Run for thirty days. Measure baseline metrics before you grant any write or containment permissions. Red-team every expansion. Attribute every action to an agent identity. Keep the audit trail append-only. And when an analyst override rate drops suspiciously low, treat it as an alert, not a win.

Part 2 of this series covers how I have it configured — profile separation, Kanban setup, HITL gates, audit trail requirements, and the specific config values that turned the architecture into a working pilot.

The agents are not coming. They are already here. The question is whether you are running them, or they are running you.

---

## Sources

1. <a id="src-1"></a>ISC2 — "Cybersecurity Workforce Study 2024" (2024). [https://www.isc2.org/Research/Workforce-Study](https://www.isc2.org/Research/Workforce-Study)
2. <a id="src-2"></a>Tilbury, A. & Flowerday, S. — "The Effects of Alert Fatigue on SOC Analysts" (MDPI Computers 2024). [https://www.mdpi.com/2073-431X/13/7/165](https://www.mdpi.com/2073-431X/13/7/165)
3. <a id="src-3"></a>Checkmarx — "2025 Trends on AI Security: How AppSec Must Evolve with the AI-Shifted SDLC" (Sep 2025). [https://checkmarx.com/learn/ai-security/2025-trends-on-ai-security-how-appsec-must-evolve-with-the-ai-shifted-sdlc/](https://checkmarx.com/learn/ai-security/2025-trends-on-ai-security-how-appsec-must-evolve-with-the-ai-shifted-sdlc/)
4. <a id="src-4"></a>Cloud Security Alliance — "Vibe Coding's Security Debt: The AI-Generated CVE Surge" (Apr 2026). [https://labs.cloudsecurityalliance.org/research/csa-research-note-ai-generated-code-vulnerability-surge-2026/](https://labs.cloudsecurityalliance.org/research/csa-research-note-ai-generated-code-vulnerability-surge-2026/)
5. <a id="src-5"></a>Digital Applied — "OpenClaw vs Hermes vs Codex CLI: 2026 Coding Agent Benchmark" (2026). [https://www.digitalapplied.com/blog/openclaw-hermes-codex-cli-coding-agent-benchmark-2026](https://www.digitalapplied.com/blog/openclaw-hermes-codex-cli-coding-agent-benchmark-2026)
6. <a id="src-6"></a>Hermes Agent Documentation — [https://hermes-agent.nousresearch.com/docs/](https://hermes-agent.nousresearch.com/docs/)
7. <a id="src-7"></a>NousResearch/hermes-agent GitHub — [https://github.com/nousresearch/hermes-agent](https://github.com/nousresearch/hermes-agent)
8. <a id="src-8"></a>OWASP Gen AI Top 10 (2025). [https://genai.owasp.org/llm-top-10/](https://genai.owasp.org/llm-top-10/)
9. <a id="src-9"></a>arXiv:2509.10540 — EchoLeak / CVE-2025-32711 (2025).
10. <a id="src-10"></a>arXiv:2601.20727 — Ojewale et al., Audit Trail Requirements (2026).
11. <a id="src-11"></a>arXiv:2602.19555 — Jiang et al., Agentic AI Attack Surface (2026).
12. <a id="src-12"></a>arXiv:2403.02817 — Nassi et al., Morris II Worm (2024).
13. <a id="src-13"></a>ACM DOI:10.1145/3759260 — Hagen et al., Automation Bias in SOCs (2025).
14. <a id="src-14"></a>Unit 42 — "AI Agent Prompt Injection Evasion Techniques" (Mar 2026). [https://unit42.paloaltonetworks.com/ai-agent-prompt-injection/](https://unit42.paloaltonetworks.com/ai-agent-prompt-injection/)
15. <a id="src-15"></a>Secure.com — "AI Didn't Just Find More Bugs, It Broke the System for Tracking Them" (2026). [https://www.secure.com/blog/cybersecurity/ai-didnt-just-find-more-bugs-it-broke-the-system-for-tracking-them](https://www.secure.com/blog/cybersecurity/ai-didnt-just-find-more-bugs-it-broke-the-system-for-tracking-them)
16. <a id="src-16"></a>GreyNoise Labs — "The PoC Pollution Problem" (Jul 2025). [https://www.labs.greynoise.io/grimoire/2025-07-30-ai-poc/](https://www.labs.greynoise.io/grimoire/2025-07-30-ai-poc/)
17. <a id="src-17"></a>Drata — "Building Smarter AI Security Research" (Apr 2026). [https://drata.com/blog/building-smarter-ai-security-research](https://drata.com/blog/building-smarter-ai-security-research)
18. <a id="src-18"></a>RedMonk — "AI Slop & the Vulnerability Treadmill" (May 2026). [https://redmonk.com/kholterhoff/2026/05/05/ai-slop-vulnerability-treadmill/](https://redmonk.com/kholterhoff/2026/05/05/ai-slop-vulnerability-treadmill/)
19. <a id="src-19"></a>CyberScoop — "AI Might Cut False Positives, But It Won't Stop the Slop" (2026). [https://cyberscoop.com/ai-vulnerability-reporting-bug-bounty-noise/](https://cyberscoop.com/ai-vulnerability-reporting-bug-bounty-noise/)
20. <a id="src-20"></a>Tines — "Voice of the SOC Analyst" (2022). [https://www.tines.com/reports/voice-of-the-soc-analyst/](https://www.tines.com/reports/voice-of-the-soc-analyst/)
21. <a id="src-21"></a>arXiv:2404.08144 — GPT-class CVE exploitation success rates (2025).
22. <a id="src-22"></a>PoisonedRAG — Zou et al., "PoisonedRAG: Knowledge Corruption Attacks to Retrieval-Augmented Generation" (USENIX Security 2025). [https://arxiv.org/abs/2402.07867](https://arxiv.org/abs/2402.07867)
23. <a id="src-23"></a>ENISA — "European Vulnerability Database (EUVD)" (live since May 13, 2025). [https://euvd.enisa.europa.eu/](https://euvd.enisa.europa.eu/)
24. <a id="src-24"></a>VulnCheck — "Does ENISA EUVD live up to all the hype?" (May 2025). [https://www.vulncheck.com/blog/enisa-euvd](https://www.vulncheck.com/blog/enisa-euvd)
25. <a id="src-25"></a>Chen et al. — "AgentPoison: Red-teaming LLM Agents via Poisoning Memory or Knowledge Bases" (NeurIPS 2024). [https://arxiv.org/abs/2407.12784](https://arxiv.org/abs/2407.12784)
26. <a id="src-26"></a>Cloud Security Alliance — *Autonomous but Not Controlled: AI Agent Incidents Now Common in Enterprises* (Apr 2026). [https://cloudsecurityalliance.org/artifacts/autonomous-but-not-controlled-ai-agent-incidents-now-common-in-enterprises](https://cloudsecurityalliance.org/artifacts/autonomous-but-not-controlled-ai-agent-incidents-now-common-in-enterprises)
---

*Last updated: 2026-06-16. Metrics and citations reflect sources retrieved through this date, including ENISA EUVD coverage. For the latest Hermes Agent configuration reference, see https://hermes-agent.nousresearch.com/docs/.*
