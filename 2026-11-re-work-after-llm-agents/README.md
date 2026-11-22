# Reverse-engineering work after LLM agents: what 2026 hiring, vulnerability and software-protection data say about the structure of the field

English | [简体中文](README.zh-CN.md)

| Item | Value |
|---|---|
| Author | [1A7432](https://github.com/1A7432) |
| Version | v1.0, 2026-11-23 |
| Sources collected | 2026-11-22 to 2026-11-23 (UTC+8) |
| Distribution | TLP:CLEAR |
| Nature | A desk review of published sources. **It contains no new measurement.** Most sources were read as the excerpts a search index returned, not as full texts. See section 2 |

---

## Summary

This note asks two questions from the point of view of industry structure, leaving technique aside: will professional reverse engineers be made redundant by LLM agents, and is there room for new work on the protection side.

- **The title is shrinking at the same rate as the security job market, not faster.** One US count of job postings puts "reverse engineer / malware analyst" at about 1,600 postings for January to September 2026, down 28.9% on the same months of 2025, against a market-wide fall of 26.4%.
- **The loss is concentrated at the entry level.** Senior-titled security postings grew 65% in the six months to March 2026 while junior-titled postings grew 5.9%. Survey data show few outright headcount cuts, and those cuts fall on the roles where people used to learn the trade.
- **Finding became cheap. Confirming did not.** Published CVE records for January to October more than doubled year on year. Bug-bounty programmes were paused or closed under invalid automated reports, and the backlog of valid, open reports on one platform grew 131% in two years. Vendors that run large-scale AI discovery describe review, not discovery, as the limit on useful output.
- **The field is adversarial, so demand is set by the other side.** Tooling that lowers the cost of analysis also lowers the cost of hiding. This is the main structural reason the function persists while its lower rungs disappear.
- **Protection vendors have started selling "LLM-resilient" products, and each one measures itself.** No independent, comparable measure of how much a given protection costs an agent was found. The published experiments agree on one behaviour that such a measure would need to capture: an agent that cannot verify its result tends to report a plausible wrong answer and stop.

The practical reading: the job title will become rarer, the ability will become more expensive, and what separates the two outcomes for an individual is whether their work sits on the producing side or the judging side.

---

## 1 Key findings

| # | Finding | Evidence | Confidence |
|---|---|---|---|
| K1 | Postings for reverse engineer / malware analyst fell 28.9% year on year in the US (Jan–Sep 2026), close to the market-wide 26.4% | One posting count [S1] | Medium (one source, one country, postings not people) |
| K2 | Entry-level demand is weakening much faster than senior demand | Posting data [S2]; workforce survey [S3]; practitioner statements [S4] | Medium |
| K3 | The bottleneck in vulnerability work moved from discovery to validation | CVE counts [S5]; bounty programme changes [S6][S7]; backlog data [S8]; vendor account [S9] | High for the direction, low for any single number |
| K4 | New role categories absorb part of the function under other titles | Posting data [S1]; national workforce report [S10] | Medium |
| K5 | Commercial protection vendors now market resistance to LLM analysis, with self-run tests | Vendor releases and papers [S11][S12][S13] | High that it is happening |
| K6 | Agents confronted with hardened binaries route around protections and may end on a confident wrong answer | One exploratory study [S14]; consistent with the author's own review data [S15] | Medium (small studies) |
| K7 | Buyers lack a common way to compare protections against agents | Stated in an industry survey for security AI generally [S16]; inferred for protection products from the absence of any independent benchmark in this search | Low to medium (an absence of evidence) |
| K8 | How long the remaining human advantage lasts | — | **Unknown. No claim is made** |

---

## 2 Scope, method and limits

**Method.** Three rounds of web search on 2026-11-22 and 2026-11-23, in English and Chinese, covering hiring data, vulnerability-discovery economics, and software protection against LLM analysis. Sources were kept if they were dated 2026 and gave either numbers or first-hand statements.

**Limits.**

- Most sources were read as the excerpts returned by the search index. Full texts were not all read, so context around a quoted number may be missing. **Check the original before citing any figure from this note.**
- Hiring figures are almost all for security work in general. Only [S1] reports the reverse-engineering title separately, and only for the United States.
- Posting counts measure what employers asked for, not employment.
- [S4] is a third-party summary of a conference panel, not a transcript.
- Vendor tests of their own products ([S11], [S13]) are marketing as well as measurement. [S13] says so in its own text.
- The search was not systematic. Sources that contradict these findings may exist and were not found.
- The author works in this area and builds agent tooling for it, which is a bias in favour of the conclusion that verification matters.

---

## 3 What the hiring data show

**The title follows the market down.** For January to September 2026, [S1] counts 235,786 US security postings, down 26.4% from 320,210 a year earlier and the lowest in four years. Within that, reverse engineer / malware analyst has 1,605 postings, down 28.9%. Penetration tester fell 45.9% and threat hunter 40.0%. AI security engineer rose 268.3%.

**The split is by seniority.** [S2] reports senior-titled postings up 65% and junior-titled up 5.9% over the six months to March 2026. In [S3], 74% of organisations say AI is changing team size or role structure, but only 16% report an actual reduction in headcount; where roles were reduced, SOC and security analysts lead (32%), then threat-intelligence analysts (26%) and incident responders (22%). The same survey finds that skills gaps overtook headcount as the main workforce problem, 60% to 40%.

**Practitioners describe the same thing.** In [S4], an instructor who has taught reverse engineering since 2015 says entry-level jobs have been disappearing, sharply in the previous six months, and points out that no juniors now means no experts later. Another panellist expects roughly the bottom half of rote work to be automated, and notes that the median capability of models rises as more work is published and trained on.

**China.** A national workforce report [S10] puts the 2025 domestic market at about 88.7 billion yuan, down 1.5%, with listed vendors' staff down about 5.1%. Of a sample of job descriptions, 40.6% mention AI skills. Its stated conclusion is that roles are being reshaped, with execution work taken over and judgement work gaining value. An industry survey [S16] argues that the commonly quoted national shortage figures come from parties with an interest in training, and that the real condition is a surplus of entry-level graduates alongside a shortage of people who combine security and AI engineering.

---

## 4 Why the function persists

**4.1 Demand is set by an adversary.** In most fields a productivity tool reduces the labour needed for a fixed amount of work. Here the amount of work is not fixed: it is produced by someone who is trying to hide something and who has the same tools. [S14] shows the defensive side of this directly (section 6). The same holds for malware, where cheaper analysis meets cheaper variation.

**4.2 The scarce step moved downstream.** [S5] reports CVE records for January to October rising from 39,638 in 2025 to 89,749 in 2026. Google paused parts of its bounty programme after a surge of largely invalid automated reports [S6]; curl ended its paid programme in January 2026 for the same stated reason [S7]. On one platform, open valid reports grew 131% between 2024 and 2026 even as average resolution time fell from 135 to 62 days [S8]. Microsoft's account of its own AI vulnerability research states that adding automated auditors increases candidate volume without increasing validated findings, and that the unit to optimise is a reproducible finding, not a report [S9]. One practitioner quoted in [S17] puts it this way: verifying an automated conclusion takes more seniority than producing the finding did.

**4.3 Some conclusions need a person to answer for them.** Attribution, forensic evidence, audit and incident liability attach consequences to a named party. [S4] records the point that an AI should not be the final accountable party for, say, a nation-state attribution. This part of the work is protected by institutions, not by technical difficulty.

**4.4 Earlier tools widened the field.** The interactive disassembler, scripting, and the decompiler each made entry easier and each was followed by more practitioners, because more targets became worth analysing [S4]. Whether this repeats is not known. The argument for it is that the volume of code nobody has read is rising quickly.

---

## 5 Why people are nonetheless displaced

**5.1 The interchangeable middle.** Work that consists of operating tools competently on familiar patterns is the first thing a model learns. The threshold moves up every year (section 3).

**5.2 The training path is being removed without a replacement.** Entry-level roles were the apprenticeship. [S17] quotes a consultant describing the result as trading a cost reduction today for a talent shortage in a few years. For people already past that stage this is protective, since fewer people follow them. For the field it is a risk.

**5.3 The title dissolves into other roles.** Part of the fall in K1 is reclassification: the same ability is hired under threat intelligence, AI security, product security or anti-abuse engineering. Counting titles overstates the loss of the function.

---

## 6 The protection side

**Vendors have moved.** One JavaScript protection vendor shipped a release in August 2026 described as targeting the inference limits of AI agents, including leading models to "prematurely claim successful deobfuscation" [S11]. A mobile protection vendor published a benchmark of an open-source obfuscator against 10 models on 200 programs: with three passes stacked, average success was 20.6% on x86-64 assembly and 8.5% on ARM64, with the best models reaching 36% on x86-64 assembly and 58% when given pseudocode [S12]. A Chinese vendor published a white paper with tests of its own product [S13].

**What the independent experiments found.** A security firm gave sandboxed agents progressively hardened ARM64 binaries and reports that it never saw an agent defeat a protection: the agents routed around each one, and switched to execution whenever static work looked unprofitable. Its proposed defence is the classical anti-tamper recipe: put the secret behind execution, bind the result to the environment instead of to a branch, and return a plausible wrong answer instead of crashing. It adds that a self-contained binary holding everything needed to produce its secret is broken by design, and that obfuscation remains a cost multiplier, not security [S14]. A separate study measured cost: one model solved 40% of 20 obfuscated challenges at an average of $2.39 per success and $4.83 per failure, and none of the tasks with multiple stacked layers [S18]. An academic system that transforms programs to make detection agents miss vulnerabilities cut detections by 52.0–60.7% across four agents [S19].

**Extraction is not exploitation.** One assessment recovered protected configuration and key material from mobile test packages with an agent, yet did not achieve token replay or device impersonation, and argues the two should be evaluated separately [S20]. This is the architectural answer: what matters is whether the extracted material is sufficient to act, which is decided by server-side design.

**The gap.** Every test above was designed and run by the party whose product or method was under test, on its own samples, with its own success criterion. [S13] notes that its results do not represent other software, models or attack settings. For security AI products in general, 36% of surveyed companies cited the lack of an evaluation system and described "demo-style success" as common [S16]. No independent benchmark that lets a buyer compare protections by the cost they impose on an agent turned up in this search.

Such a measure would need at least three numbers per protection level: success rate, cost in money and time, and the rate of confident wrong answers. The third is the one current tests mostly omit, and [S14] suggests it is where protections actually work. In the author's own review data, 40 of 94 findings from small-model workers on a real sample were rejected, most of them with correct addresses and a wrong characterisation [S15], which is the same failure seen from the analyst's side.

---

## 7 Discussion

A three-layer reading fits the sources:

| Layer | Work | Direction |
|---|---|---|
| Entry | Operating tools by procedure on common samples | Largely taken over; the way in is closing |
| Middle | Competent but interchangeable analysis | Under steady pressure as the model median rises |
| Top and edge | Validating conclusions, targets outside the training distribution, building the systems that make agents useful, answering for results | Scarcer and better paid |

Two things would change this picture. If model limits fall away within months instead of years, the top layer narrows; [S4] records a senior researcher saying he cannot tell which it will be. And if attackers adopt agents at scale, demand on the defensive side rises; [S5] finds that observed exploitation has not yet followed what demonstrations show to be possible, and cannot tell whether defenders absorbed the load or attackers have not yet scaled.

---

## 8 Planned follow-up

A small public measurement of protection strength against agent cost: a fixed set of pure-computation programs with known answers, each built at several protection levels with open-source protectors only, attacked with one fixed agent configuration, reporting success rate, cost, and the rate of confident wrong answers per level. Commercial protectors are excluded because their licences commonly restrict publishing test results. No date is set.

---

## Sources

| # | Source | Date |
|---|---|---|
| S1 | CyberSN, *Cybersecurity Jobs Report, 2026 through September*. https://cybersn.com/resources/reports/cybersecurity-jobs-report (the figures for penetration tester, threat hunter and AI security engineer were taken from a news summary of this report) | 2026-10 |
| S2 | AI Workforce Consortium analysis of Cornerstone and Indeed data, as reported in [S17] | 2026-08 |
| S3 | SANS Institute and GIAC, *2026 Cybersecurity Workforce Research Report*, press summary. https://www.sans.org/press/announcements/sans-research-cybersecurity-talent-shortage-narrative-wrong-real-crisis-what-your-team-doesnt-know-starting-ai | 2026 |
| S4 | Summary of the RECON 2026 closing panel on AI and reversing. https://hexdigest.dev/e/4939-recon-2026-the-ai-reversing-panel-are-we-all-powerful-or-out-of-a-job | 2026 |
| S5 | *SoK: Assessing the Evolution of LLM Capability in Software Vulnerability Research*. https://arxiv.org/abs/2611.26601 | 2026-11-19 |
| S6 | CSO Online, *Google's bug bounty pause highlights growing AI vulnerability triage challenge*. https://www.csoonline.com/article/4231109/googles-bug-bounty-pause-highlights-growing-ai-vulnerability-triage-challenge.html | 2026-10-06 |
| S7 | Inspectiv, *The State of AI in Bug Bounty Hunting*. https://www.inspectiv.com/articles/the-state-of-ai-in-bug-bounty-hunting | 2026-07-17 |
| S8 | HackerOne, *Security Research Report 2026, 10th Edition*. https://www.hackerone.com/report/hacker-powered-security | 2026 |
| S9 | Microsoft Security Blog, *3 lessons from frontier AI vulnerability research*. https://www.microsoft.com/en-us/security/blog/2026/10/07/3-lessons-from-frontier-ai-vulnerability-research/ | 2026-10-07 |
| S10 | 《AI时代网络安全产业人才发展报告（2026）》, summary published by DBAPPSecurity. https://www.dbappsecurity.com.cn/content/details6004_198127.html | 2026-09 |
| S11 | Jscrambler, *Code Integrity LLM-Resilient*. https://jscrambler.com/blog/jscrambler-enhances-code-integrity-llm-resilient/ | 2026-08-13 |
| S12 | Tkachenko and Erdogan, *Obfuscation Resilience Against LLM Deobfuscation* (IEEE CSR 2026). https://gencer.no/wp-content/uploads/2026/08/2026.IEEECSR.pdf | 2026 |
| S13 | Virbox, 《Virbox Protector 对抗 AI 自动化逆向分析白皮书》. https://shell.virbox.com/articles/保护能力/virbox-protector-ai-automated-reverse-engineering-defense-white-paper/ | 2026-08-13 |
| S14 | Quarkslab, *Defeating AI-Assisted Reverse Engineering (or at Least Trying To)*. https://blog.quarkslab.com/defeating-ai-assisted-reverse-engineering-or-at-least-trying-to.html | 2026-08-20 |
| S15 | 1A7432, *An LLM worker harness for IDA reverse engineering*, section 5. [../2026-11-llm-re-worker-harness/](../2026-11-llm-re-worker-harness/) | 2026-11-22 |
| S16 | 数说安全, industry research report beginning 《重塑网络安全：网络安全智能化产…》 (title truncated in the search excerpt; subtitle 《从安全助手到安全智能体》). https://pdf.dfcfw.com/pdf/H3_AP202604281821651603_1.pdf | 2026-04 |
| S17 | CSO Online, *5 ways AI is reshaping the cybersecurity job market*. https://www.csoonline.com/article/4224019/5-ways-ai-is-reshaping-the-cybersecurity-job-market.html | 2026-09-19 |
| S18 | Elastic Security Labs, *The Cost of Understanding: LLM-Driven Reverse Engineering vs Iterative LLM Obfuscation*. https://www.elastic.co/security-labs/threat-command/llm-reversing-vs-llm-obfuscation | 2026-04-21 |
| S19 | Tian et al., *Hardening JavaScript Programs against LLM-Based Vulnerability Detection*. https://arxiv.org/abs/2611.25718 | 2026-11-19 |
| S20 | Cryptomathic, *Obfuscation vs AI: How much protection does your app really have?* https://www.cryptomathic.com/blog/obfuscation-vs-ai-how-much-protection-does-your-app-really-have | 2026-10-30 |
