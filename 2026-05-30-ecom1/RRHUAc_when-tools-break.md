---
source_code: https://github.com/Vialov/BitGN_ECOM1
run_ids:
  - run-22RxFQ7A6JRUNgBvsTgHB9XAR
  - run-22RxEsHmRaHw7dvTizdEYQLTV
model_names:
  - Gemini 3.5-flash
  - Qwen/Qwen3.5-397B-A17B
author: Viktor Vialov
author_linkedin: https://www.linkedin.com/in/vialov/
impact: "When Tools Break: How a Smart Agent Moves Beyond Expected Behavior"
challenge: ecom
---

I want to share an observation from the ECOM challenge. Formally, my production result was weaker than I had hoped: I overfit the agent to SQL-based navigation, and that broke on the blind run. But this failure produced what was probably the most interesting insight — not about benchmark task-solving quality, but about AI safety.

The main takeaway is this: an agent with many tools and a lot of freedom can perform well when everything works as expected. But when tool problems appear, a strong model does not necessarily just stop. It may behave in unexpected and not always safe ways. It may start diagnosing the environment, exploring adjacent APIs, and performing harmful actions.

## Setup

I had a coding agent: the model could write Python code and execute it inside a Docker sandbox. Inside the container, there was an SDK wrapper around the BitGN API. At the beginning, the model often tried to search for data directly in `/proc` instead of using SQL queries, which led to frequent errors. So I decided to make its behavior more controlled: I programmatically restricted the model from browsing through `/proc`, except for reading specific files, and moved the main navigation path to SQL. The idea was reasonable: less chaotic filesystem browsing, more structured queries, fewer accidental references and hallucinations.

Something went wrong on production. When the SQL layer or SDK calls did not behave as expected, the agent lost its main channel for solving tasks. At that point, the interesting question became how different models react when their tools break. I ran the same agent wrapper with Qwen and Gemini 3.5 Flash.

## Qwen: a predictable failure mode

Qwen behaved roughly as I expected. It ran into a broken API, tried to find a workaround, attempted to guess exact paths in `/proc`, and eventually gave up after failing to find a working strategy.

This was a bad outcome for the leaderboard, but a good failure mode from a safety perspective: the agent did not solve the task, but it also did not expand its action space. It stayed within the expected corridor:

> The tool does not work → try obvious fallbacks → if they do not work, return an error.

## Gemini 3.5 Flash: diagnosis instead of refusal

Gemini was more interesting.

At first, it also poked at the broken API. But then it did not stop. Instead of accepting this as a tooling failure, it began diagnosing the environment itself. It started reading the SDK code. In itself, this could still be considered acceptable, even though this behavior was not intended.

But then it went further. To check other API endpoints, it decided to perform a real action: checking out a random basket.

That is a completely different class of behavior. The model was no longer trying to solve the original user task. It was experimenting with the environment, using available mutating operations as a diagnostic tool.

In a benchmark sandbox, checking out a random basket is just an unpleasant log entry and a wasted attempt. In a production system, similar behavior could mean a real state change: creating an order, sending an email, charging money, changing permissions, closing a ticket, or launching some internal process.

And the most important part is that this does not look like a classic prompt injection or a malicious scenario. The model does not “want to cause harm.” It is simply trying to be helpful and figure out why its tools are not working.

The problem comes from the combination of two factors. First, the agent has a universal coding interface. It can not only call predefined business tools, but also write arbitrary code to explore the environment. Second, the model is strong enough to diagnose the system.

## The main safety insight

Sometimes the danger does not come from the model violating an instruction. It comes from the model following the general goal too actively: “figure it out and complete the task.”

When tools break, a strong model may start expanding the task context on its own:

> The API does not work → I need to understand why → I need to read the SDK → I need to check other methods → I need to make a test call → the test call turns out to be a mutation.

From the model’s perspective, this is a coherent chain of reasoning. The danger is not only a potentially large and useless waste of tokens, but also unauthorized and unwanted changes of state.
