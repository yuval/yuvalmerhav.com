+++
title = "Self-Improving Agents: Rewrite the Harness or Grow the Knowledge?"
date = 2026-10-12
draft = false
+++

## Intro

I attended COLM 2026 last week in SF. I'm starting a blog series about some of the talks and posters I saw there. I took a bunch of notes, and writing them down helps me take it all in. Putting it here in case others find it useful.

I'll start with a topic that's been getting a lot of buzz lately: harness self-improvement, aka self-modifying software and harnesses, auto research, etc. etc. In this post I focus on only two papers - there are many more, but these are the ones I happened to look into. My time is limited, and I usually prefer going deep on a couple of works over doing a breadth survey. Micro type of guy vs. macro. The two approaches are also very different, so they complement each other nicely.

- [Meta-Harness](https://yoonholee.com/meta-harness/) ([code](https://github.com/stanford-iris-lab/meta-harness)). A coding agent searches over the harness code itself, with full access to past code, scores and traces.
- [Knowledge-Centric Self-Improvement](https://arxiv.org/abs/2607.19592) ([code](https://github.com/recursive-knowledge/KSI)). Agents write and curate a shared *knowledge base* that gets fed to future agents.

![Meta-Harness vs. Knowledge-Centric self-improvement loops](/self-improvement-loops.svg)

Both are worth reading. I spend most of the post on what I think actually holds up, and where the evaluation is weaker than the headline numbers suggest.

## Meta-Harness: End-to-End Optimization of Model Harnesses

Chelsea Finn gave the first keynote, and a big part of her talk was about Meta-Harness, which was also presented at the conference. She then extended it to robotics, but surprisingly the approach wasn't that different. 

Before diving in, here's a simple way of looking at this topic. We all use coding harnesses such as Claude Code, OpenCode, Pi, etc. Each can be paired with different models. Both matter, but here we focus on the harness.
Let's think of the harness as the code that wraps a model, including the prompts, tool definitions, memory implementation, etc. Developers have been working hard to build these harnesses (with the help of AI, of course), and at some point we've all made some harness improvements by hand - added a skill, chagnging the system prompt, etc. When we talk about self-improvement, we usually mean starting with some harness and improving it further. But in the extreme case, you can start almost from scratch - you write 100 lines of Python that run the main inner agent loop, call a model, maybe give it a couple of tools, and that's it. This simple harness likely won't perform very well on hard tasks, but that's where self-improvement can come in and build a much more capable harness for you. I haven't seen people attempt this - they always start with an existing good and already optimized harness to some degree, but I'm sure we'll see new harnesses pop up that take this approach.

The key ingredient is having verifiable data for the task we're optimizing the harness for. For example, for a coding task we need coding problems and a way to check whether a generated solution is correct. For a classification problem, we need labeled examples. For math, we need math problems and a way to verify the solutions, similar to coding. Etc.

Once you have this data, you can feed tasks to the agent, let it solve them, check whether it got them right, and if not, run a process that improves the harness. That process is what this work is about.

Meta-Harness defines two loops:

1. **Inner loop**: a harness plus a frozen model (e.g., Haiku 4.5) runs on tasks, producing execution traces and solutions. A separate evaluator scores the solutions (e.g., accuracy, test pass/fail).
2. **Outer loop**: a separate proposer (Claude Code with Opus 4.6, max reasoning) looks at past harnesses and their results, then writes a new harness. This repeats many times.

By "past harnesses" we mean previous iterations. We already know that coding agents work very well with local data in files. Meta-Harness builds on that and saves each harness candidate (one version of the harness that the proposer wrote and that got evaluated) in its own folder, with its source code, scores, and full execution traces: every prompt, tool call, and model output. Nothing is summarized. A single evaluation can produce up to 10M tokens, way more than fits in context, so the proposer navigates it with `grep`, `cat`, etc.

The main idea is to give a coding agent everything and impose almost nothing else. Claude Code decides which past candidates to look at, which failure to diagnose, and whether to make a small edit or rewrite the whole thing. Their results show that having the full history matters, and the proposer actually references 20+ prior candidates. If it didn't, you could just give the agent a summary of the latest results and throw away the rest.

### Results

They evaluated on three main benchmarks:
1. Retrieval-augmented math
2. Online text classification
3. Agentic coding (TerminalBench-2)

In addition, and maybe the most interesting result, they ran experiments on unseen tasks. More on this later.

Each benchmark has a different setup, and it's important to follow each one to understand what the results mean. I'm skipping online text classification - check the paper if you're interested. I'll start with retrieval-augmented math because it's the simplest to understand.

#### Retrieval-augmented math

Before solving a new math problem, it helps to show the model a few solved problems, but only if they're actually useful for the new one. In fact, they show that random few-shot hurts performance. Here the harness picks which solved examples, from a corpus of ~535K, to show the model before it tries a new olympiad problem. The proposer ran 40 iterations, producing 109 candidate harnesses, each scored by how well GPT-OSS-20B (the solver) did on 250 search-set problems. The best one was then evaluated on 200 unseen IMO-level problems, with GPT-OSS-20B plus four models not used during search (GPT-5.4-nano, GPT-5.4-mini, Gemini-3.1-Flash-Lite and Gemini-3-Flash). The paper's abstract calls all five held-out, but GPT-OSS-20B was the search model.

| Method | Avg over 5 models |
|---|---|
| No retrieval | 34.1 |
| Random few-shot | 32.2 |
| BM25 | 37.5 |
| Dense retrieval (k=5) | 38.1 |
| Meta-Harness | 38.8 |

The final harness works in two steps. First, it guesses the problem's topic (combinatorics, geometry, number theory, or other) using simple keyword checks. Then it retrieves similar solved problems with BM25, using a slightly different rule per topic, e.g., how many examples to pull and whether to favor harder problems.

A neat detail: the proposer usually builds each new harness by editing an earlier one, so the search grows a few separate lines of descendants. One line ended up best at geometry, another at combinatorics, and the proposer merged them on its own, taking each topic's rule from the line that handled it best.

How much does it help? Not much over standard retrieval. On average it beats BM25 by 1.3 points and dense retrieval by 0.7, and dense retrieval actually does better on both Gemini models. The authors' main argument is consistency: Meta-Harness improves over no retrieval on all five models, while dense retrieval and random few-shot hurt some of them.

#### TerminalBench-2

This one is a bit more interesting. [TerminalBench-2](https://www.tbench.ai/) is a popular benchmark of 89 hard tasks that agents solve in a terminal. Each task runs in its own Docker container and comes with a human-written solution and tests that verify the result. The tasks are inspired by real workflows, e.g. configuring legacy systems, reimplementing research papers, or fixing security vulnerabilities. The benchmark keeps evolving - most labs currently report results on Terminal-Bench 4.0, and I think there's a new version in the works. Work presented at conferences is usually ~6 months behind in terms of models and benchmarks.

They optimize the harness on all 89 tasks. What do they evaluate on, you may ask? The same tasks. They actually discuss this: they frame it as a discovery problem and note that the approaches they compare against do the same thing, so at least in this comparison it's fair. They do run a separate experiment on generalization to held-out datasets, but it's not coding. More on it in a bit.

The search starts from two seed harnesses: Terminus 2 and Terminus-KIRA, which is built on top of Terminus 2 with native tool calling and smarter completion checks. In each iteration, the proposer edits a harness's Python code and the new version runs on all 89 tasks. The results are actually surprising. Compared to KIRA, the main change was a new ~80-line function that snapshots the environment (OS, languages, package managers, `/app` contents, memory) with a 15-second timeout and appends it to the initial prompt as an `[Environment Snapshot]` block.

| Model | Terminus-KIRA | Meta-Harness |
|---|---|---|
| Haiku 4.5 | 33.7 | 37.6 (#1 among Haiku agents) |
| Opus 4.6 | 74.7 | 76.4 (#2 among Opus agents) |

Meta-Harness scored 76.4%, second only to ForgeCode at 81.8%. The authors mention that they couldn’t reproduce ForgeCode’s result from its public code alone. They suggest its leaderboard score depends on components that aren’t in the published repo.

**Generalization (Table 5).** All these improvements are nice, but not that practical if they don't generalize to unseen tasks. The math section above already showed some generalization (held-out problems and unseen models), but held-out *datasets* were tested only for online text classification, the benchmark I skipped above. TerminalBench-2 has no held-out evaluation at all. They take the best harness found on three classification datasets (LawBench, Symptom2Disease and USPTO-50k) and evaluate it on 9 unseen classification datasets, such as 5-way Amazon review rating prediction and 3-way financial sentiment classification. Meta-Harness outperforms the next best method, [ACE](https://arxiv.org/abs/2510.04618) (Agentic Context Engineering), by 2.9 points on average across all 9 datasets. They conclude that the harness captures generally effective strategies for text classification rather than overfitting to the specific datasets used during search.

My take: the snapshot idea looks real and generic, but showing that it works on unseen coding tasks would make it more appealing. Another point worth mentioning: when they swap the inner-loop model from Haiku 4.5 to Opus 4.6, the baseline improves significantly and their gains are smaller. This is expected, and it might be that with Opus 5.5 they wouldn't see gains at all. But that's OK, since there's still a lot of value in using cheaper and faster models and improving the harness so it matches the results of more expensive models.

### Appendix Highlights

**D: practical tips** is the most actionable section if you want to try this:

- The skill file given to the proposer mattered *more* than iteration count or population size. They ran several short 3–5 iteration runs just to debug the skill.
- The skill should constrain outputs and forbidden actions, not prescribe how to diagnose problems.
- Build the search set from examples the baseline gets wrong, small enough for ~50 full evaluations.
- Log everything in consistent, grep-friendly JSON. A small CLI for listing the frontier and diffing runs helps.
- Keep evaluation outside the proposer.


## Knowledge-Centric Self-Improvement
- [Knowledge-Centric Self-Improvement](https://arxiv.org/abs/2607.19592)

This paper presents a very different approach to the same problem. Instead of rewriting code, agents build a shared knowledge base (KB) across many attempts. The agents are disposable: we don't optimize the harness, we optimize the KB. Basically, agents build "memory" together for the next rounds, and that memory keeps evolving over time.

The KB is a SQLite store behind an MCP server. The setup runs in **generations** (10 per benchmark). The gist is pretty simple - in each generation, an agent tries to solve a task defined in `TASK.md`. Each agent also receives `MEMORY.md`, which is the KB distilled into something the agent can work with. Once a task is solved correctly, it leaves the pool, so tokens aren't wasted revisiting it.

How is `MEMORY.md` generated? I'll keep it high level - check the paper for the exact details. You can think of the KB as a forum. Each agent working on a task contributes a post about what it learned solving that task, and in addition, a similar learning that's relevant across tasks, so other tasks can benefit from it. Agents are also forced to query the forum first, so they know what already exists before posting something new.

There's also some guidance for what agents should post. For example, a **cross-task post**, which is a claim meant to help *other* tasks, must name a concrete primitive (an API call, error type, invariant, test-runner flag) rather than a vague "approach", and quote where it showed up. For example: "In Rust's loop-based propagation, snapshot values before the loop and compare after; Java's synchronous propagation can compare in place."

Between generations, one LLM call (no agent involved) turns the forum posts into the next generation's bundles. This is the distillation step. Before a fresh agent starts, the system builds a seed package: the per-task bundle for its task, the shared cross-task bundle, and the attempt history for that task. All of it is rendered into the `MEMORY.md` in the agent's workspace, which the agent reads with its normal tools. There's no KB API and no retrieval by the agent during solving. Selection happens *before* the agent runs.

We can think of the per-task forum as retry memory, and the cross-task forum as the source of reusable knowledge.

### Results

For coding, they evaluated on Terminal-Bench 2, Polyglot, and SWE-Bench Pro. I'm glossing over the results quickly here - see the paper for more details and benchmarks.

Like with Meta-Harness, the experimental setup is important. For SWE-Bench Pro, for example, they run the loop on a sample of 50 tasks and report the final success rate. Recall that the agent makes up to 10 attempts (generations) per task. They report the success rate after the run completes - there are no separate unseen tasks they evaluate on. SWE-Bench Pro has many more tasks, so I don't see a clear reason not to evaluate on held-out ones, even if it's a small sample to keep things cheap. For Terminal-Bench 2, they use the same setup as Meta-Harness and evaluate on the benchmark's 89 tasks. They also use 3 seeds and report the average and standard deviation, which is important because these benchmarks are small and results can vary from run to run - something I don't think Meta-Harness did.

The task-solving agents they compare are the Claude Agent SDK with Haiku 4.5 and the OpenAI Agents SDK with GPT-5.4-mini using medium reasoning. On Terminal-Bench 2, they report a pass rate of 43.8% ± 3.4, compared to 37.6% for Meta-Harness. Two caveats here. First, the 43.8% is cumulative: a task counts as solved if *any* of the up to 10 generations solved it. Meta-Harness and the leaderboard report the pass rate of a single fixed harness, so the two numbers aren't directly comparable. Second, the TB2 numbers for Meta-Harness are taken from what its authors reported, not reproduced (the other baselines - DGM, HyperAgents, GEPA, and OpenEvolve - were rerun). Since these benchmarks are fairly small, it's hard to draw conclusions from a few percentage points of difference. The more important takeaway is the ideas the papers present.

#### Section 4.4: Held-Out Transfer

Now the interesting part. Does the memory help on tasks it has never seen, with no retries, i.e., one attempt per task? They ran an experiment to find out, but only on Polyglot (coding) and ARC-AGI-1 (abstract reasoning). They used a held-out test set of 20 tasks for each benchmark and showed improvements over zero-shot without memory - on the order of going from about 2 to 4 solved tasks. As expected, which model is used for the solver agent and for the memory builder has a significant impact on the results. The runs with memory used an extra LLM call before each task to make a plan that includes the memory. I think it'd be better to add a similar "plan" step without the memory to the zero-shot runs, so we can isolate the planning gains from the memory gains. Also, the authors point out that 7 of the 20 Polyglot tasks are the same exercise as a "training" task in another language (e.g., Go vs. Java). 


## Related Work

For reference, here are a few related works these papers compare against: ACE, DGM, HyperAgents, GEPA, and OpenEvolve. I asked Claude to write a TL;DR for each of them, and here's what it came up with:

**ACE (Agentic Context Engineering)**: [arXiv 2510.04618](https://arxiv.org/abs/2510.04618)
ACE improves an LLM system by growing a "playbook" in its context instead of changing weights or code. Three LLM roles share the same model. The Generator solves a task using the current playbook. The Reflector reads the trajectory, using execution feedback and labels when available, and extracts concrete lessons; it also tags which playbook bullets helped or hurt. The Curator turns those lessons into small "delta" additions, which are merged into the playbook by simple non-LLM code, with embedding-based deduplication. The key argument is that rewriting the whole context each step causes "context collapse": in one AppWorld run, the context shrank from 18K tokens to 122 in a single step, and accuracy fell below baseline. Appending itemized bullets avoids this. It works offline (building a system prompt from a training set) or online (updating memory while running through the test stream). It reports +10.6% on AppWorld agents and +8.6% on finance tasks over baselines like GEPA and Dynamic Cheatsheet.

**DGM (Darwin Gödel Machine)**: [arXiv 2505.22954](https://arxiv.org/abs/2505.22954)
DGM is a coding agent that improves by editing its own Python codebase: its tools, prompts, and workflow. It keeps an archive of every agent version ever created. Each iteration picks a parent from the archive, weighted toward high performers but also toward less-explored ones. The parent diagnoses its own benchmark failures and writes a code change to itself, and the child is evaluated and added to the archive. Unlike the original theoretical Gödel Machine, it validates changes by running benchmarks rather than proving them correct. Because the task is coding, getting better at the task also means getting better at modifying itself. It reported gains on SWE-bench (about 20% → 50%) and Polyglot (about 14% → 31%), with discovered improvements like better file-editing tools and patch validation.

**HyperAgents**: [arXiv 2603.19461](https://arxiv.org/abs/2603.19461)
HyperAgents extends DGM by making the self-improvement procedure itself editable. It combines a task agent, which solves the target task, and a meta agent, which modifies both itself and the task agent, into a single editable program. In DGM, the logic that decides how to improve was fixed. Here it lives in the same codebase that gets rewritten, so the system can improve how it improves. The loop is DGM-style: each generation, the meta agent inspects the current agent code and the performance history of all prior agents, then proposes code edits that are evaluated in an isolated Docker container, with improvements kept in an evolutionary archive. It comes from Meta, Oxford, and Jeff Clune's group, the same lineage as DGM.

**GEPA (Genetic-Pareto)**: [arXiv 2507.19457](https://arxiv.org/abs/2507.19457)
GEPA is a prompt optimizer for multi-step LLM programs, built into DSPy. It runs the program on a few examples, then has an LLM read the full traces and any feedback, such as error messages or evaluator comments, and write a natural-language diagnosis plus a revised prompt for one module. It keeps a pool of candidate prompts and selects parents from a Pareto frontier: any candidate that is the best on at least one training example survives. This preserves diverse strategies instead of greedily keeping only the top average scorer. It can also merge two good candidates. The headline claim is that it beats RL fine-tuning (GRPO) and MIPROv2 with up to 35× fewer rollouts. The ACE paper criticizes it for "brevity bias": converging on short, generic prompts.

**OpenEvolve**: [GitHub](https://github.com/algorithmicsuperintelligence/openevolve)
OpenEvolve is an open-source reimplementation of DeepMind's AlphaEvolve, not a paper. It does evolutionary search over code. You mark the parts of a program that can be changed and supply an evaluator that scores it. Each step, it samples parent programs from a database organized for both quality and diversity (MAP-Elites-style, with separate "islands" that evolve in parallel). An LLM then proposes a diff, the result is scored, and the program is added back to the database. It's built for problems with a single clean score, such as algorithms, math constructions, or kernels. In the two papers covered here, it serves as a baseline for different targets: Meta-Harness used it to evolve harness code, while Knowledge-Centric Self-Improvement used it to evolve only the agent's system prompt.