---
source_code: https://github.com/rbpp3042/bitgn-ecom-api
run_ids:
  - run-22RyE5w3Y2vAsKpstb134F8WW
model_names:
  - deepseek-v4-flash
author: Egor Kolosov
author_github: https://github.com/rbpp3042
author_telegram: "@blue_tape"
impact: Mechanical harness with a task classifier, linters, and preflight checks wrapping DeepSeek v4 Flash — business logic that does not need an LLM is handled without one, the model handles the rest.
challenge: ecom
---

# BitGN ECOM1: Mechanical Harness on a Cheap Model

I built an agent on DeepSeek v4 Flash — a cheap open-weight model — and wrapped it in a mechanical harness: a task classifier, a set of linters, and preflight checks. Any business logic that can be done without an LLM, is done without an LLM. Result: 51.4/100 tasks, ~46th place, $0.28 for a full run on DeepSeek Flash.

The result is nothing special, but I picked up some important lessons along the way and want to share them with the community.

![Architecture diagram](res/PrtzSE_architecture.png)

---

## Background

I started with the dev benchmark: sent Claude Code Opus 4.7 in, looked at the tasks, understood the format. Claude handled everything fairly confidently — but I wasn't interested in just maximizing the score with a frontier model. The interesting question was different: **can you build a working agent on an open-weight model?**

I tried gemma-4-31b-it and nemotron-3-super-120b-a12b — both were too slow through OpenRouter and lost context quickly. DeepSeek Flash worked right away: fast enough, smart enough for structured tasks. I switched it to the direct DeepSeek backend and stuck with it.

---

## How Does It Work?

The agent is split into two layers: mechanical and language.

**The mechanical layer** runs without an LLM. It includes:
- a task classifier — determines the task class from the task text (checkout, refund, fraud audit, etc.)
- a set of linters — validate the agent's response before the final submission
- preflight checks — run before any mutating action

**The language layer** is the model itself: reads documents, reasons, makes decisions.

### Progressive Disclosure

The core idea: keep the main instruction file (`AGENTS.MD`) as short as possible. I aimed for ~100 lines. Initially it was ~300 lines — with instructions covering every case at once. The model performed worse: important rules got lost in the noise.

The solution: the task classifier determines the class before the model is ever called, and a task-specific chunk of instructions is appended to the base prompt. The model gets a short general context plus concrete rules for the current task — nothing more.

### Linters

Linters are mechanical checks that run after the model has drafted a response, but before the final submission. They verify:
- whether the OUTCOME token format is correct
- whether the response includes the required document references
- whether the agent is citing a file it never actually opened

If a check fails, the agent receives a specific hint describing the problem and can correct its response. This catches formal errors mechanically, without spending model reasoning on them.

For checkout, there's an additional **preflight**: before calling the tool, the agent mechanically checks inventory — is there enough stock, is the basket valid, are there any conflicts? If not, it informs the model, which then corrects its action. Result: **0.95 on the checkout task class** with a cheap flash model.

### Caching

Requests are structured so that as much context as possible hits the cache. The full competition run on DeepSeek Flash cost **$0.28 for 100 tasks**.

---

## Models

- Main solver: DeepSeek v4 Flash (direct backend, not OpenRouter)
- No classifier or evaluator model — classification is mechanical (regex + keyword rules)
- All models are open-weight

---

## What Broke at the Prod Launch

The production environment turned out to be different in several ways — some of which I hadn't anticipated.

**New tools.** The dev environment didn't have the refund tools (`bitgn-refund`). The agent didn't know about them → an entire task class dropped out completely.

**Different data structure.** Paths to data files (orders, employees, products) were randomized in production. `baskets/` could become `carts/<cust>/`, `employees/` — `staff/`, `products/` — `catalog/<Brand>/`. I was partially prepared for this — I added dynamic file search instead of hardcoded paths.

**Different identifier format.** In dev, IDs looked like `cust_001`; in production — `cust-0039`. All mechanical checks that relied on a specific format became inert and silently passed everything without checking. A trivial mistake that a five-minute smoke test before the paid run would have caught.

---

## Where the Agent Lost the Most

### Refund (−4 tasks)

The agent knew about the `bitgn-refund` tool from the system prompt. But at the slightest uncertainty it chose the "safe" path — respond with `OUTCOME_NONE_UNSUPPORTED` ("no such tool"), rather than call the tool.

The problem wasn't that the tool wasn't mentioned. The problem was the absence of an explicit prohibition: *if the tool exists and is callable — never respond UNSUPPORTED*. The model needs a contract, not a hint.

### Archive/TSV (−8 tasks) — Two Different Defects

An interesting case: two fundamentally different problems were hiding under the same label.

**First — a linter bug.** The fraud audit task on an archive file: find fraudulent rows, calculate the total, cite the specific rows in the format `file.tsv#row=ROW_ID`. The agent correctly found the rows and calculated the sum. But at submission, the linter rejected the response: the path `file.tsv#row=X` didn't match `file.tsv` in the list of opened files. Correct answer = 0 due to a technical bug in the harness. Not the model's fault.

**Second — new task types.** Some task classes appeared only in prod and weren't present in dev — the agent was completely unprepared for them. An unforeseeable surprise that can't be addressed without knowing the prod task set in advance.

### Discount (−3 tasks)

The model applied the baseline rule "discounts are manager-only" and refused without checking for exceptions. In `/docs/discounts/addenda/` there were delegating addenda — documents that could authorize a specific employee for a specific basket. The model wasn't looking for them.

Lesson: instruction priority matters. "Check exceptions first" must come before the base rule — not after it.

### Fraud Detection

Task: find fraudulent transactions in the payments table. The agent found one suspicious cluster and stopped — considered the task done. But the table contained several independent clusters, and the correct answer required finding all of them.

The working solution: flag a transaction as fraudulent only with ≥2 independent signals + scan the entire table globally rather than stopping after the first find.

Lesson: read the task literally. "Find fraudulent transactions" is not "find one incident."

---

## Lessons

**Design the agent for a changeable environment.** The most expensive lesson of the competition: a mechanical harness calibrated for dev broke in several places at once on prod. For a competition agent, you need to run diagnostic probes at the very start of each run across all key environment components — API, filesystem structure, available tools, identifier formats. Confirm that everything works as the agent code expects — not as it worked yesterday in dev.

More importantly, design the agent so that its core components can be easily swapped when the environment changes: data paths, ID formats, tool list. Then adapting to a new environment means editing a config, not rewriting logic.

**The size and wording of AGENTS.MD matter.** When the instruction file was ~300 lines, the model frequently looped on the same steps, didn't try alternative approaches, and lost track. After trimming to ~100 lines — it started applying nearly everything written there. Length isn't the only factor: every rule needs to be worded as tightly as possible. Example: was `"use bitgn-refund for refunds"` — became `"if the bitgn-refund tool exists and is callable, never respond UNSUPPORTED"`. The second version works; the first doesn't.

**The correct answer and the proof of that answer are two separate engineering tasks.** The BitGN scorer evaluates not just the OUTCOME token but also the list of references (refs): which files the agent drew on. Correct answer without correct refs = 0.

In practice: the model reaches the right conclusion but either forgets to cite the required file, or cites a file it never opened. The linter catches this mechanically and returns a hint — and the agent reformulates. But the linter can also fail: in the archive fraud case, it became the source of the problem itself, blocking a correct answer because of a path format mismatch.

Conclusion: "giving an answer" and "proving an answer with references" are two separate requirements, each needing its own infrastructure. And that infrastructure itself needs to be tested.

---

## Mechanical Harness vs. Universal Agent

The competition exposed an interesting tension.

For a competition environment, a mechanical harness is a design mistake: the environment is randomized, hardcoded assumptions break, the agent struggles to adapt. A more universal agent that explores the environment at startup has a clear advantage here.

In real business, it's different. The environment doesn't change overnight. New tools appear gradually — and can be added explicitly. Data structure changes happen within a deploy — and that's the moment for explicit harness adaptation. The mechanical approach wins here: it's faster, cheaper, more predictable. A self-adapting universal agent in production is a source of unpredictability, not an advantage. And all new tasks and environment changes are better escalated to HITL first anyway — and only automated after that.

A competition with an adversarial environment is a specific kind of problem. I would stick with the same mechanical approach on a real project.

---

*Code: [github.com/rbpp3042/bitgn-ecom-api](https://github.com/rbpp3042/bitgn-ecom-api) · Telegram: [@blue_tape](https://t.me/blue_tape)*
