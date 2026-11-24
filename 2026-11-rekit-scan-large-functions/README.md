# Asking questions of a function too large to decompile: range scans and run-time strings in a read-only IDA API

English | [简体中文](README.zh-CN.md)

| Item | Value |
|---|---|
| Author | [1A7432](https://github.com/1A7432) |
| Version | v1.0, 2026-11-25 |
| Measured | 2026-11-25 (UTC+8) |
| Distribution | TLP:CLEAR |
| Code | [1A7432/retools](https://github.com/1A7432/retools), `tools/rekit/` (a snapshot, not maintained) |
| Nature | A short engineering note, a follow-up to [the worker harness note](../2026-11-llm-re-worker-harness/). One binary, one function, single measurements |

---

## Summary

The read-only IDA API described in the earlier note (`rekit`) was built around function cards and pseudocode. It had nothing to offer for a control-flow-flattened function of about 470,000 instructions: the function cannot be decompiled, and a database-wide search for a constant ran out of time. This note records the two calls added for that case and what they measured.

- **`scan` answers four questions about one function or address range without decompiling it**: what it calls and how often, which immediates it uses, where one immediate is used, and what value is in a given register at each call of one target.
- **The range is read once.** The first question took 2.9 s for 466,107 instructions. Later questions about the same range are answered from the kept listing and took under 10 ms.
- **`decode` recovers strings that are not in the file** by emulating the one-time initialisers that build them. Cards show them in a separate field, marked as decoded.
- **Before using either, count prologues and returns.** The function had been written down as "several functions that IDA merged by mistake". It has one prologue and one return. That wrong note stayed in the project's documents until someone asked whether all 2 MB really was one function.

The binary is not named. Function names, addresses, constants and strings in the examples are replaced; the counts and timings are real.

---

## 1 The case

The target is one function in a large ARM64 binary:

| Property | Value |
|---|---|
| Size | about 2 MB of code, 466,107 instructions |
| Shape | control-flow flattened; one prologue, one `RET`, one stack frame |
| Decompilation | not practical |

What the analysis needed from it was narrow and factual: which system calls it makes, which helper writes each output field, and where a few dozen constants seen in its output come from. None of that needs pseudocode.

Two things went wrong before any tooling was involved, and both are cheap to avoid:

1. **The size was explained away.** An earlier note said IDA had merged several functions. Nobody had checked. Counting prologue patterns and return instructions in the range takes seconds and settles it.
2. **The search was too wide.** `rekit.find(imm=...)` searches the whole database. In this one it hit its 20 s budget and returned a partial result (one hit, which was inside the function in question). The question was about one function all along.

These questions were first answered with one-off IDAPython scripts. `scan` packages those scripts, and the figures below were measured again with it; they agree with the scripts.

---

## 2 `scan`

All four forms take a function (all its chunks) or a range `[ea, end)`.

| Call | Question | Result here |
|---|---|---|
| `rekit.scan(f)` | What does it call, by count? | 9,554 direct call sites to 113 targets, 108 indirect calls. `malloc` ×4,409, `memcpy` ×1,136, `bzero` ×768, `free` ×356 |
| `rekit.scan(f, show="imms", min=0x10000)` | Which constants does it use? | 3,466 distinct immediates of 0x10000 or more |
| `rekit.scan(f, imm=K)` | Where is one constant used? | the instruction addresses, in this range only |
| `rekit.scan(f, to="_syscall", reg="w0")` | Which system calls does it make? | 31 sites, four distinct numbers: ×22, ×4, ×3, ×2 |

The call profile alone says something about the function: more than four thousand allocations and a thousand copies in one body is code that assembles and encodes data, with the logic inlined rather than called.

The last form was the most useful. The same call with a different target and register listed the 339 sites of one helper together with the small integer passed in `W1` at each. That one result gave the full set of field numbers the function can write, and showed that 30 numbers never seen in its output have no code at all: they are reserved, not conditional.

### 2.1 How the register value is found, and when it is not

For each call site, `scan` reads backwards along the straight line into the call and takes the last write to the register. It reports the value only when that write moves an immediate. Otherwise it shows `?` with the reason:

| Reason | Example |
|---|---|
| The write is not an immediate | `MOV X0, X19` |
| A jump target lies between the write and the call | another path may bring a different value |
| An earlier call left its result in the register | the value is a return value |

This is deliberately not data-flow analysis. In flattened code the dispatcher makes real data flow expensive to follow, and a wrong value presented as certain is worse than a `?`. The sites marked `?` are read by hand with `rekit.disasm`.

It follows registers only. Arguments pushed on the stack (32-bit x86) are not found, and on x86 a write to a sub-register (`al`) is not seen as a write to `eax`.

---

## 3 `decode`: strings built at run time

Some strings never appear in the file. A compiler splits a short literal into integer constants, or an obfuscator stores it encrypted and decodes it on first use. Either way the code has the same shape: read a flag byte, skip if set, build the string, set the flag.

A function card now looks for that shape (a writable byte the function both reads and writes), emulates each initialiser from the flag read to just after the flag write, and lists what was written:

```
"decoded": {"total": 8, "top": [
    "0x81f2b4c \"FetchRemotePolicyForDevice\" | init 0x2f4dc4-0x2f4e38",
    "0x81f2a10 \"policy_manager.cc\" | init 0x2f50d8-0x2f5124 | loop"], ...}
```

In the function this example is modelled on, 8 initialisers were found and all 8 ran to the end. One of the strings was the name of a command the function handles, which no string search could have found.

Three rules keep this honest:

- **Decoded strings are kept apart from literals.** They are what the program's own code writes when run on the file's initial memory. A report quoting one should say "decoded".
- **Fragments are never joined by guesswork.** A constant whose bytes are printable is shown with its text (`0x6d65526863746546 "FetchRem"`), as one fragment. The joined string appears only when the initialiser could be run.
- **Failures are counted, not hidden.** Initialisers that did not run to their end are reported as a number, and the ordinary string list is left as it was.

Not covered: statics guarded by `__cxa_guard_acquire` (tried; too few ran to the end), strings built on the stack with no flag, and decode routines that take arguments.

---

## 4 What this did not solve

- **`scan` tells you where, not why.** It located a CRC loop and the table it uses. The exact input to that CRC was still not reproduced after about 1.66 million variants were tried offline.
- **A count is not a reading.** "31 system calls, 22 of them the same number" directed the manual reading to a handful of sites. Every conclusion about what the function does came from reading those sites and from checking against its output.
- **Tools do not stop a worker from overreaching.** Small-model workers set on the same function returned correct addresses with a wrong description of the surrounding condition more than once. The earlier note's finding stands: facts from tool output are mostly reliable, characterisations need review.

---

## Appendix: Where the numbers come from

| Number | Source |
|---|---|
| 466,107 instructions; 9,554 / 113 / 108; call counts; 3,466 immediates; 31 sites and their split; 339 sites | `rekit.scan` output on the database, 2026-11-25, one run |
| 2.9 s first scan, under 10 ms afterwards | wall-clock time around the calls in the same session |
| 20 s budget on the database-wide search | the note returned by `rekit.find(imm=...)` in the same session |
| 8 initialisers | `rekit.decode` output for one function |
| 30 reserved numbers, 1.66 million variants | analysis records of the same project, not published |
