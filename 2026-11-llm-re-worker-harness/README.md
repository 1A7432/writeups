# An LLM worker harness for IDA reverse engineering: a guard proxy, a findings ledger, and one blind benchmark run

English | [简体中文](README.zh-CN.md)

| Item | Value |
|---|---|
| Author | [1A7432](https://github.com/1A7432) |
| Version | v1.0, 2026-11-22 |
| Data collected | 2026-11-13 to 2026-11-22 (UTC+8) |
| Distribution | TLP:CLEAR |
| Code | [1A7432/retools](https://github.com/1A7432/retools) (a snapshot, not maintained) |
| Nature | An engineering record with small-sample measurements. **It is not a rigorously controlled benchmark study.** See section 2 for the limits |

---

## Summary

This note describes a local harness in which a large orchestrating model splits an IDA reverse-engineering job into narrow tasks, hands them to cheap small-model workers, and verifies what comes back. It reports what the harness enforces mechanically, what one blind benchmark run measured, and how often worker findings survived review on a real sample.

- **Enforcement is done by a proxy, not by the prompt.** The worker never talks to the IDA MCP server. It talks to a guard proxy that exposes 4 of the server's tools, checks every call against a fixed read-only API (`rekit`), enforces the call budget, and logs each call. Any internal error makes the proxy exit, which leaves the worker with no tools. After each run an audit compares the worker's event stream with the proxy log. A non-IDA tool call, or a result that bypassed the proxy, voids the run.
- **Blind benchmark: 58 questions on 6 stripped, obfuscated ELF binaries, 53 of them machine-graded.** Workers alone scored 47/53. Fixing one tool bug that the benchmark exposed gave 48. Adding four tools aimed at the remaining error classes gave 52, with the same worker model and the same question text. The whole workflow with orchestrator verification also scored 52; its one miss was an answer spelling the grader did not accept.
- **Five of the six initial worker errors were interface problems.** One was a tool bug, three were missing tools, and one was a transcription slip that a mechanical back-reference removes. Only one was an analysis error, and it stayed wrong after the tools were added.
- **On a real sample, 40 of 94 worker findings were rejected on review (43%).** Rejection is per finding: one wrong detail rejects the whole finding. In most rejected findings the listed addresses and API names were correct, and the error was in an attribution, a number base, or the stated purpose of a function. "The evidence address was seen in a tool result" is therefore a weak signal and does not replace review.
- **Cost.** A narrow task costs a worker somewhat over 100,000 tokens. An open-ended task ("work out what this program does") costs 1.2–2.4 million.

Each number is from a single run. The benchmark is self-made and small, there is no baseline of the large model working alone, and the reviewer on the real sample was a stronger model that could see the worker's answer.

---

## 1 Key findings

| # | Finding | Evidence | Confidence |
|---|---|---|---|
| K1 | Limits on the worker are enforced mechanically by an MCP proxy: 4 tools exposed, only a fixed read-only API allowed through, a hard call budget, full logging, and exit on any internal error | Code and tests (`guard.py`, `guardproxy.py`; 158 offline checks) | High |
| K2 | On the 53 machine-graded blind questions: workers alone 47; 48 after one tool bug fix; 52 after four more tools; whole workflow 52 | Grader output, one run each | Medium (small sample, single runs) |
| K3 | Of the 6 first-round errors, 5 trace to the interface (1 tool bug, 3 missing tools, 1 transcription) and 1 is an analysis error | Per-question manual attribution, plus a rerun after the tools were added | Medium |
| K4 | On a real sample, 94 worker findings were reviewed: 54 confirmed, 40 rejected (43%) | Ledger records | Medium (the review was not blind) |
| K5 | In rejected findings the addresses and API names are mostly correct; the errors are in attribution, number base, or characterisation | Reading the rejection notes | Medium |
| K6 | An open-ended task costs more than ten times the tokens of a narrow one | Run records | High |
| K7 | Whether this approach beats the large model working alone | — | **Not measured. No claim is made** |

---

## 2 Scope, method and limits

**What the system is.** A set of local tools. The orchestrator is a coding-agent session of a large model. The workers are small models run headless (deepseek-v4.1-flash in these measurements). Target architectures are x86, x86-64 and ARM64. The host is macOS on Apple Silicon.

**Where the code came from.** The author set the design, the constraint rules and the acceptance criteria. Most of the code was written by coding agents under those constraints, with the author reviewing and deciding. In this note, "orchestrator verification" means verification by the orchestrating model, not line-by-line human verification, unless stated otherwise.

**How the blind benchmark was run.**

- The question set `obf1` has 6 stripped ELFs (x86-64 and ARM64) and 58 questions. Three programs are pure obfuscation (control-flow flattening, opaque predicates, dead code, mixed boolean-arithmetic, a bytecode virtual machine). Two are cryptographic (SM4, SM3 and SHA-256 variants; SM2 encryption and signing; AES-CTR; with decoy keys, masked constants and a hidden curve point). Every program is pure computation with no malicious behaviour.
- Setting and answering were separated. The sources, answers and grader live in a directory the answering session does not read, and the answering orchestrator had not read them before grading.
- 53 questions are graded by script. 5 explanation questions are graded by hand against a checklist of required points.
- The questions and binaries are in the repository under `bench/obf1/public/`. The sources, answers and grader are not published, because the set could then no longer be used blind.

**How the real-sample review was done.** The sample is a 14 MB 32-bit Delphi PE. Every finding workers produced on it was entered in the ledger, then checked one by one against the IDA database, and the verdict was written back to the ledger. The reviewers were the orchestrator and a second, stronger model; the second model did nearly all of it.

**Limits.**

- Each configuration was run once. There are no repeats and no variance estimates.
- The question set is self-made and small, and it leans toward cryptography and virtual machines. The gain from 47 to 52 comes from tools added in response to known misses and a rerun on the same questions, so overfitting to this set is possible. The four tools have not been validated on a second set.
- Two baselines are missing: the large model answering alone, and the small model using IDA directly without this system.
- The reviewer on the real sample could see the worker's answer, so the review was not blind. The review itself was not spot-checked again.
- Two more question sets (`obf2`, `jsvm1`) have been built. This note reports no scores for them.
- After the code moved to its own repository, all offline tests pass, but no worker task has been rerun with a real model.

---

## 3 Design

The starting observation: when a general-purpose model does reverse engineering, most of its mistakes are interface mistakes, not reasoning mistakes. A small model guesses IDAPython APIs wrong. Reading back an xref list of several hundred thousand characters floods its context. Copying a long hex string by hand drops characters. So the interface is fixed, the output is compressed, and everything that can be checked mechanically is handed to a machine.

### 3.1 A fixed read-only API

`rekit` is a read-only module inside IDA. It provides function cards, pseudocode, xrefs, call trees, search and disassembly. Its conventions:

- Every call returns a dict, and addresses are always hex strings.
- Every call has a hard character budget. When output is truncated, the reply must state the total and how to fetch the rest.
- An error comes back as "error plus hint", never as a traceback.

Measured: a function card with code is about 50% fewer characters than "raw pseudocode plus a flat xref list", and a triage card without code is about 90% fewer. An xref query on one hot function went from 935,000 characters to 1.4 KB.

### 3.2 The guard proxy

The MCP server the worker sees is a proxy, and the proxy starts the real IDA MCP server.

| Mechanism | What it does |
|---|---|
| Tool visibility | Exposes only 4 tools (open database, list databases, execute Python, look up reference). No save and no close |
| Database | Only the one database named in the task can be opened |
| Call content | The content of every execute-Python call is checked. In strict mode only `rekit` calls pass |
| Budget | The proxy enforces the cap on the number of calls. Near the cap, results carry a note with the number of calls left |
| Logging | The request and the result of every call are written to disk in full |
| Failure direction | On any internal error the proxy exits, and the worker then has no tools |

The guard is a proxy and not a hook in the agent framework because of the failure direction. A hook lets calls through when it crashes, times out, or cannot read its configuration, which makes it unfit to be the only line of defence. A proxy is independent of the framework the worker runs on and carries over to another backend unchanged.

### 3.3 Contract, grounding check and audit

- **Contract.** A task has six fields. A result is `status / summary / findings / unknowns / needs`, and each finding carries evidence addresses and a confidence level. Results are capped in characters and in number of findings.
- **Grounding check.** An evidence address must have appeared in the task or in the worker's own tool results; otherwise the finding is marked as not grounded. Numbers below 0x1000 do not count as addresses, so that a struct offset cannot ground a finding.
- **Audit.** After each run, the worker process's event stream is compared with the proxy log. A non-IDA tool call, or a result obtained around the proxy, voids the whole run.
- **Exact-value back-references.** In its answer the worker writes a reference to the result of a specific call, and the tool substitutes the original value. Any long hex string left in the answer that cannot be found in any tool output is listed.

### 3.4 The ledger

Each database has an append-only ledger. A worker's finding enters as "unverified". The orchestrator or a reviewer marks it confirmed or rejected and records the reason.

One rule matters most: **function cards show only verified notes.** Otherwise an unverified worker finding would be cited back as evidence by the next worker, and errors would reinforce themselves.

### 3.5 Supporting tools

| Tool | Purpose |
|---|---|
| Function-level emulation (Unicorn) | A fresh instance per request. Every import is intercepted. Initialisation functions can be run before the target function |
| Whole-program behaviour reports | PE and ELF are emulated inside a macOS sandbox; the child process checks that the sandbox is in effect before touching the sample. An ELF can also be really executed in a container with no network, no mounts and no capabilities |
| Calculator | A purpose-written restricted interpreter with no `eval`, no files, no imports and no attribute access. It has built-in reference implementations of hashes, block ciphers and elliptic curves whose standard constants can be overridden |
| Crypto-constant matching | Compares constant tables in the database against reference tables entry by entry. On a near match it reports the indexes and values that differ |

---

## 4 Blind benchmark results

### 4.1 Scores

| Configuration | Machine-graded (53 questions) |
|---|---|
| Workers alone | 47 |
| Same, with the one task affected by a tool bug rerun on the fixed tool | 48 |
| Same, plus four tools (constant matching, whole-program runs, calculator, exact-value back-references) | 52 |
| Whole workflow (workers plus orchestrator verification) | 52 |

By program:

| Program | Workers alone | With four tools | Whole workflow |
|---|---|---|---|
| t1_easy | 8/8 | 8/8 | 8/8 |
| t2_medium | 7/8 | 8/8 | 8/8 |
| t2_medium_arm64 | 2/2 | 2/2 | 2/2 |
| t3_hard | 8/9 | 8/9 | 9/9 |
| t4 | 10/13 | 13/13 | 13/13 |
| t5 | 12/13 | 13/13 | 12/13 |

The last two columns both total 52, but they miss different questions. The whole workflow's miss is a curve name written in a form the grader's accepted list does not contain; the answer and the reference answer name the same curve. On the 5 hand-graded explanation questions, both workers alone and the whole workflow covered every required point.

### 4.2 Attribution of the six first-round errors

| Question | Cause | Attributed to |
|---|---|---|
| The accepting input of the medium program | The memory image used for emulation was exported one byte short per segment, so the correct input was judged as rejected | Tool bug |
| Which opcode handlers never execute | Read "never executed" as "unreachable from the dispatch table" | Worker analysis |
| Which SHA-256 constant was changed | Compared 64 constants by eye and missed one hex digit | Missing tool |
| The full output line for a given input | Computed an HMAC step by step with function-level emulation and ran out of budget | Missing tool |
| Recover the plaintext from a ciphertext | Needed a decryption computation, and the worker had no means of computing | Missing tool |
| A 180-byte raw output | Dropped 3 characters while copying a long hex string | Transcription |

After the tools were added, the four tasks containing misses were rerun with the same question text. The workers used the new interfaces on their own initiative: the calculator 27 times, whole-program runs 7 times, constant matching once, and 21 value references. Total time for the four tasks fell from 2879 s to 1793 s. Tokens were about level (5.35 million against 5.66 million).

The question still answered wrong is the "never executed" one. The worker moved from "all of them can execute" to listing three handlers, and missed the handler for invalid opcodes. That is analysis that was not careful enough, not a missing tool.

### 4.3 Open-ended analysis

Given only "work out what this program does", the five programs have 27 required points between them, and one round of workers covered about 23. The three errors:

- Treating a SHA-256 with altered constants as the standard algorithm, and blaming the emulator when results did not match.
- Reading a rotate-left by 4 as an add of 4.
- Missing a dead function that nothing references.

Orchestrator verification caught all three, by comparing the constant table, by reimplementing the virtual machine in another language and recomputing, and by checking cross-references.

### 4.4 Cost

| Phase | Workers | Tokens | Time per worker |
|---|---|---|---|
| Open-ended analysis | 5 | 10.5 million | 9–16 min |
| Targeted questions | 13 | 12.57 million | 2–16 min |

An open-ended task costs 1.2–2.4 million tokens, against somewhat over 100,000 for a narrow one. The gap comes from the larger number of calls, each of which rereads an ever longer context. The great majority of the tokens are cache reads.

---

## 5 Review results on a real sample

On the Delphi PE, workers produced 95 findings (87 in strict mode, 8 in free mode). 94 of them have a review verdict:

| Verdict | Count | Share |
|---|---|---|
| Confirmed | 54 | 57% |
| Rejected | 40 | 43% |

Rejection is per finding: one error in a finding rejects all of it. Most rejection notes have the form "this part is right, that part is wrong". The wrong parts fall into a few kinds:

| Kind | Example |
|---|---|
| Overstated characterisation | A function that reads taskbar-button data from another process was called a process-injection routine. The APIs it listed all exist, but nothing writes memory and there is no remote thread |
| Wrong attribution of an argument or field | Stack arguments passed to a text-measuring function were attributed to a different call |
| Wrong number base | The code uses decimal 732 (0x2DC); the finding wrote 0x732 |
| Library function misidentified | The runtime library's exception-finalisation routine, and a dictionary's collision-count property, were taken for business logic |
| Incomplete list claimed complete | Claimed a full inventory of network-related functions but left out the only caller of `getpeername` |
| Wrong ordering | A check that happens before the receive call was described as happening after it |

Every one of these findings was "grounded": its evidence addresses really had appeared in tool results. The grounding check can therefore rule out addresses invented from nothing, but not a wrong statement made while looking at a real address. The confidence levels workers report for themselves also run high.

This is the data behind the ledger rule that unverified findings do not reach function cards. At a 43% rejection rate, letting unverified findings flow back to later workers would mean that close to half of the "known facts" carry an error.

---

## 6 Discussion

**Make a limit something the worker cannot do, not something it is told not to do.** "Read-only" and "do not call other tools" in a prompt are requests. A proxy that exposes 4 tools and exits on error is a fact. The audit then checks again from the other side, and a mismatch voids the run.

**Find out whose error it is before drawing a conclusion.** Five of the six first-round misses did not need a stronger model. They needed one export bug fixed, three tools added, and a value reference. The total score alone suggests "the small model is not good enough"; per-question attribution suggests the opposite.

**A small model's facts are more reliable than its characterisations.** The benchmark and the real sample agree. Call chains, reference counts and string locations, which can be read straight off tool results, are mostly reliable. Summaries of the form "this function does X" tend to overreach. Tasks should ask for facts and leave characterisation to the verifier.

**Verification is part of the cost and cannot be skipped.** A 43% rejection rate means an unverified ledger is unusable. Verification used a more expensive model, and that cost was not metered separately here. This note therefore cannot answer whether the approach is cheaper than the large model working alone.

**Keep tasks narrow.** Open-ended tasks cost more than ten times as much, and their coverage of required points (about 23 of 27) is no better than the accuracy on narrow tasks.

---

## 7 Known problems

- Workers are slow. Even a tiny question takes tens of seconds to minutes. For a single lookup, the orchestrator is faster using the read-only API directly.
- A worker will not ask when the task wording is ambiguous. State exactly what is being counted, and ask for the call chain.
- Free mode blocks write operations by name only. It is not a sandbox and can be bypassed deliberately, so strict mode is the default.
- The IDA MCP server saves a database automatically when the last session releases it. The tools changed nothing, but the modification time of the database file changes.
- Real execution in a container is not an isolation boundary fit for hostile malware.
- Constant matching cannot find constants computed on the spot from a formula (SM4's CK, AES's Rcon) or derived from other data with SIMD.
- Whole-program emulation is of limited value for large GUI programs: the Delphi sample here stopped during runtime-library initialisation.

---

## Appendix A: Reproduction

```sh
git clone https://github.com/1A7432/retools && cd retools
./test                      # offline tests; suites that need Docker or the worker backend are skipped
ls bench/obf1/public/       # the 6 binaries and questions.json
```

The answers and the grader for the blind benchmark are not in the repository, so the scores in section 4 cannot be recomputed from outside. Only the system itself and the questions can be reproduced. The ledger behind section 5 contains analysis conclusions for a real sample and is likewise not published.

## Appendix B: Where the numbers come from

| Number | Source |
|---|---|
| 47, 48, 52, 52 and the per-program split | The grader's output on four answer sheets, one run each |
| Attribution of the six misses; about 23 of 27 points | Records from the day of the blind test (2026-11-14) |
| Call counts, times, tokens | Worker run records |
| 95 findings, 94 reviewed, 54/40 | The sample's ledger, counting the last verdict on each finding, as of 2026-11-22 |
| 50%, 90%, 935,000 characters to 1.4 KB | The Measurements section of `docs/DESIGN.md` in the repository |
