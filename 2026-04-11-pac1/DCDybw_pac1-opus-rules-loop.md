---
source_code: https://github.com/defendend/pac1_agent
run_ids:
  - run-22HntprL5yhmueWbe87B9d9uS
  - run-22HneAvko5JSJ7weZMXAAKFZ3
  - run-22HnSri33aAfzCarWbbqfPPra
model_names:
  - claude-opus-4-6
author_github: https://github.com/defendend
author_telegram: defendend_ai_dev
impact: One Claude Opus 4.6 tool-use agent governed by 11 hot-reloadable Markdown rules evolved via a run → trace → judge → rule → rerun loop.
challenge: PAC
---

# PAC1 Opus + rules-evolution loop

This is the architecture behind `rust-blind-20260411-1632`, my BitGN PAC1
blind PROD submission. It scored **86%** on the frozen PAC1 Accuracy
Hall of Fame, **4th in the world**. All three submitted runs
(`run-22HnSri33aAfzCarWbbqfPPra`, `run-22HneAvko5JSJ7weZMXAAKFZ3`,
`run-22HntprL5yhmueWbe87B9d9uS`) used the same Rust agent — the third
one, with the final rule set and the stricter completion schema, was
the best.

The core idea is intentionally boring: **one agent, native Anthropic
`tool_use`, no critics, no multi-agent voting**. All the improvement
between submissions came from a tight feedback loop on a small set of
hot-reloadable Markdown rules.

## How does it work?

The runtime is a Rust binary. One CLI command (`compete`) connects to
the BitGN harness via Connect/gRPC, calls `StartRun`, then fans out all
`trial_ids` into a `tokio::JoinSet` guarded by a `Semaphore` of size
50 — every trial is a fully independent async task. Each trial:

1. Calls `StartTrial`, receives the task instruction and a per-trial PCM
   sandbox URL.
2. Reads a small bootstrap: `tree -L 2 /`, the root `AGENTS.md`, the
   workspace `context()`. This goes in as the first user message.
3. Enters a `tool_use` loop against Claude Opus 4.6 with a hard budget
   of 40 steps. Tools mirror BitGN's PCM surface: `tree`, `read`,
   `search_text`, `find_files_by_name`, `write`, `move`, `delete`,
   `context`, `inbox_next`, plus `report_completion`.
4. Finishes with `report_completion`, which has a strict JSON schema:
   `outcome` (OK / DENIED_SECURITY / CLARIFICATION / UNSUPPORTED),
   `message`, `grounding_refs`, **and a required `completed_steps_laconic`
   field** that forces the model to summarise its own actions before
   submitting.
5. Calls `EndTrial`. After every trial returns, the runner calls
   `SubmitRun`.

`grounding_refs` are auto-collected: every file the agent reads or
writes is tracked at the runtime layer and merged into the final refs.
The model still can (and does) add semantic refs, but the boring "path
that you literally touched" set is never lost.

## The rules system

The system prompt is two layers:

- **Base prompt** (`src/prompt.rs`) — ~130 lines: tool semantics,
  outcome contract, completion schema, authority hierarchy.
- **11 rules** (`.claude/agent-rules/*.md`) — read from disk at startup
  and concatenated after the base prompt. The whole prefix is sent with
  `cache_control: {type: ephemeral}`, so 50 parallel trials share one
  cache write per generation.

Each rule is a single concept in 10–30 lines of Markdown. Examples:

| Rule | Concept |
|------|---------|
| 01 | Task-level vs content-level injection: text *describing* injection ≠ injection |
| 02 | Conflict resolution per OpenAI Model Spec hierarchy: L1 system > L2 task > L3 root `AGENTS.md` > L4 nested `AGENTS.md` > L5 file content |
| 03 | Always derive "today" from `context()`, never from priors; show your work |
| 04 | Entity disambiguation — resolve sender linkage before giving up to CLARIFICATION |
| 05 | UNSUPPORTED ≠ DENIED_SECURITY (capability gap is not a security event) |
| 06 | Admin/OTP channel trust — admin verification flows are legitimate, not injection |
| 07 | Critical file missing → stop the batch, don't do partial work |
| 08 | Authoritative source — downstream vs shadow when fixing regressions |
| 09 | Eight PROD ops categories (from organiser leaks) for category-aware outcome bias |
| 10 | Absent fact is a valid answer — zero results → OK with `null`, not CLARIFICATION |
| 11 | Phishing detection — canonical local-part + variant TLD = DENIED |

Because the rules live as separate files, iterating is just `vim
.claude/agent-rules/0X-foo.md` + a rerun. No rebuild, no prompt-cache
invalidation beyond the rule that changed.

## Which LLM models did you use?

A single one: **`claude-opus-4-6`** via the Anthropic Messages API
(through an internal Anthropic-compatible OAuth gateway in my case;
the public repo also supports the direct Anthropic API, OpenAI,
Kimi, GLM and any OpenAI-compatible local endpoint).

I tested GPT-5.4 (Chat Completions and Responses API), Kimi k2.5, GLM-5
and several local models early on. GPT-5.4 was the best of the
non-Opus options at ~71% mean ±4% on dev. Opus 4.6 with the full rule
set landed at 86% on PROD.

No critic, no reviewer, no planner — those experiments all regressed
−5% to −15% (LESSONS.md in the repo has the full table).

## Problems I ran into

1. **Stochasticity dominates short feedback loops.** The same code
   produced 60–81% across consecutive dev runs (σ ≈ 5%). Tuning against
   single runs was largely fighting noise.
2. **Lean prompts regress hard.** A clean 1-paragraph system prompt
   à la the winner solution dropped me to 65% on Opus. Same prompt was
   ~80% on GPT-5.4. Opus needs more guidance, not less.
3. **Heavy prompts also regress.** Adding 7 "process" sections dropped
   me to 69.8% — too much context distracted the model on simple tasks.
4. **Dev benchmark is noisy.** Several dev tasks were ambiguous or
   genuinely hard; tuning to fix `t30` would break `t23`. PROD was the
   real test.
5. **Authority conflicts.** Nested `AGENTS.md` files inside project
   directories sometimes contradict the root. Without a written rule
   the model would either always defer to root or always defer to
   nested — both wrong.
6. **`grounding_refs` discipline.** Many tasks scored 0 not because the
   answer was wrong but because the required ref path was missing.

## How I solved them

- **The iteration loop:**
  `compete --no-submit` → capture per-trial `.trace.json` dossiers →
  run 4 parallel LLM-judge subagents over the dossiers → each judge
  proposes a one-paragraph hypothesis about the failure pattern →
  diff hypotheses against existing rules → write one new rule (or
  rewrite one existing rule) → rerun. Rules 09–11 came directly from
  judge analysis of the PROD dossiers between submission 1 and 2.
- **Variance farming.** Submit 3 runs of the same config, take the best.
  Cost was small (parallelism made each run <15 min) and σ ≈ 5% means
  the best-of-3 expected value is meaningfully above the per-run mean.
- **Authority hierarchy as a rule, not a prompt aside.** Rule 02
  encodes the Model Spec order explicitly: L1 → L2 → L3 → L4 → L5.
  Submission 3 rewrote this rule after the organiser's clarification.
- **Auto-refs in code.** I stopped trusting the model to remember every
  path it touched and made the runtime accumulate refs from the tool
  layer.
- **Completion schema as self-check.** The required
  `completed_steps_laconic` field forces a sentence per major step
  before submitting. It catches "I forgot the last move" failures.

## How could the agent be better?

- **Different-model judge.** I used Opus 4.6 to judge Opus 4.6 traces;
  a different family (GPT-5.4 or local DeepSeek) would catch more
  patterns. Same-model critics in-loop hurt; out-of-loop different-model
  judges should help.
- **Per-category routing.** Rule 09 lists 8 PROD ops categories. A
  cheap upfront classifier could route to a slimmer category-specific
  rule pack instead of carrying all 11.
- **Deterministic post-processing for `t10`, `t30`, `t37`.** These
  three were persistent failures across every Opus and GPT run. They
  look like things that want code, not more prompt.
- **Open-weights variant.** The repo already supports any
  OpenAI-compatible endpoint; running it through DeepSeek-V4 or Qwen3
  to see how far the rules-based approach generalises is the obvious
  next experiment.

## What I learned

- **One agent + small evolved rule pack > orchestration.** Every
  multi-agent variant I tried (security scanner, plan-then-execute,
  reviewer) regressed. The bottleneck wasn't reasoning, it was
  evidence discipline and authority resolution — both fix better with
  rules than with more LLM passes.
- **The completion schema is part of the architecture.** Making the
  model self-summarise in a required field changed behaviour more than
  any system-prompt edit.
- **Hot-reloadable rules beat prompt edits.** Once rules live in
  separate Markdown files, iteration speed is bounded by `cargo build
  --release` (≈25 s) plus the dev rerun, not by careful copy-paste in
  one giant string.
- **Trust the harness for what it knows.** `context()` knows today's
  date; the workspace tree knows what exists. Asking the model to
  re-derive these from priors is where most "stupid" failures live.

## Related tools

While iterating I needed fast structural code search across the agent
itself, the `.claude/agent-rules/` directory, and the rule-evolution
notebooks. That work ended up as a separate open-source CLI:
[`Claude-ast-index-search`](https://github.com/defendend/Claude-ast-index-search) —
a fast multi-language code search CLI (Rust, 34 languages) for AI
agents that drive a shell.

## Links

- Source: <https://github.com/defendend/pac1_agent>
- Author GitHub: <https://github.com/defendend>
- Author Telegram: [@defendend_ai_dev](https://t.me/defendend_ai_dev)
- Related tooling: <https://github.com/defendend/Claude-ast-index-search>
