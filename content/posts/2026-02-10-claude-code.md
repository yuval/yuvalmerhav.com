+++
title = "Claude Code: My Setup"
date = 2026-02-10
draft = false
+++

  
## Intro

The nice thing about Claude Code (CC) is that it works great out of the box. This is how tools should be. But there's also a lot of customization available, with new features dropping constantly, and it can get pretty hairy and confusing.

Most of what I discuss here is available in my repo [yuval/dotfiles](https://github.com/yuval/dotfiles), so you can go straight there and ask your favorite coding agent for a summary. I keep this up to date, so it’s worth a look even if you’re reading this long after the publish date (Feb 10, 2026; CC (running Opus 4.6) and Codex (running GPT 5.3) are the SOTA coding agents). 

I'm not going to talk about my overall workflow, just my setup and why (and when) I customize Claude. For the most part, I prefer CC to do what it knows and not add too much mental burden, but I do find it useful to tweak things as needed.

## 1. Letting Claude Run Faster

CC and Codex with the latest models are slow (sorry gemini- and cursor-cli, haven't tried you yet). Making things faster is always a big win. You can always use a faster model, but that's rarely worth it for anything important. 

Interestingly, as I'm writing this, CC released "fast mode" - faster Opus 4.6 responses at a higher cost per token. It costs significantly more and isn't included in standard subscriptions. But for people working on "serious" products, I'm sure it's worth it. 

### Permissions

This is probably the most important thing for reducing latency. I often work in `accept-edits-on` (I switch modes with `shift-tab` often depending on the task) but never with `--dangerously-skip-permissions`, as most of my work is on my company's codebase, DB, etc. I know some people use sandboxes or VMs but I haven't tried that yet. 

Instead, I make sure my permission lists are up-to-date so CC doesn't bother me with requests to read a directory, run safe commands, or do web searches (I whitelist domains as I go). Here's my [settings.json](https://github.com/yuval/dotfiles/blob/main/claude/settings.json) which has pre-approved tool patterns I update frequently.

## 2. Making myself Work Faster

CC lets you enable skills, commands (which seem to be replaced by skills), agents, plugins, and hooks. I try not to over-engineer it. I add things when I notice I keep doing something slowly too often. While it sounds like a lot, you can run each of these as a command to see what you're currently using. For example, if I run `/agents` I get:

```
│ Agents                                                                                                                      │
│ 7 agents                                                                                                                    │
│                                                                                                                             │
│ ❯ Create new agent                                                                                                          │
│                                                                                                                             │
│   User agents (/Users/yuval/.claude/agents)                                                                                 │
│   shipit · inherit                                                                                                          │
│                                                                                                                             │
│   Built-in agents (always available)                                                                                        │
│   Bash · inherit                                                                                                            │
│   general-purpose · inherit                                                                                                 │
│   statusline-setup · sonnet                                                                                                 │
│   Explore · haiku                                                                                                           │
│   Plan · inherit                                                                                                            │
│   claude-code-guide · haiku
```

### Custom Agents

I only have one user agent across repos: [shipit](https://github.com/yuval/dotfiles/blob/main/claude/agents/shipit.md). I commit and push many times a day, and doing it manually or asking CC ad-hoc usually works just fine, but having a dedicated agent that's consistent, fast and just works 95% of the time was a no-brainer. This could also be a skill (more on skills later), as conceptually the line between the two isn't always clear. In CC, sub-agents spin up their own context window with a clean system prompt, which is a better fit for shipit since it doesn't need the current conversation. Shipit's entire workflow is driven by git commands: `git status`, `git diff --cached`, etc. It generates the commit message from the diff, not from conversation context.

Tip: Give your agent a color so it's clear when it's running. 

### CLIs > Copy-Pasting

I briefly explored MCPs (specifically for context as CC often makes assumptions based on out-of-date knowledge), as I found myself copying and pasting documentation. It's slow and the CC docs also recommend referring Claude to a file rather than pasting. I had the popular [context7](https://github.com/upstash/context7) plugin/MCP for example but I found the `webfetch` tool usually good enough with the right instructions for my needs. 

Many users, me included, prefer using CLIs since CC is already a CLI expert and it works well without eating a lot of tokens usually. It's very easy to understand and reproduce if needed. The ones I use often:
- `Postgres`: I created a read-only user for our database and gave CC [instructions about psql](https://github.com/yuval/dotfiles/blob/main/claude/rules/database.md). Since the DB schema is already in the codebase, it gets the context it needs before querying. Initially I was doing it manually - asking CC for an SQL query, execute it, then paste the results or point CC to a file with the results. 
- `Github` (gh) / `Gitlab` (glab) are obvious ones.
- `Frontend`: I don't do much frontend but when I do [playwright-cli](https://github.com/microsoft/playwright-cli) from Microsoft is handy so that I don't need to provide CC with screenshot and provide details. (fyi, there's also a new [chrome](https://code.claude.com/docs/en/chrome) integration)
- I still do most AWS / GCP things on their portals but I'm sure that soon I will just let CC use the aws/gcp clis for most things (navigating the GCP portal is a nightmare)

**hooks**. I don’t have any CC hooks defined right now. If I were to add one, it would likely focus on safety enforcement around potentially destructive commands like rm. My current safeguards against destructive operations mostly live in memory so they aren’t strictly enforced. I hope I don't regret that...

### Skills

Skills are probably the most talked-about feature. They are simple, and you can find many online (Anthropic has a [skills repo](https://github.com/anthropics/skills)). While the main idea behind skills is not about speed, they often help with that. I only have two custom skills.

1. *The "Todos" Skill*. The first one is a tiny skill that is specific for a repo I have where I save my notes and it's been getting pretty big. 

```
# .claude/skills/todos/SKILL.md

description: List work todos
model: Haiku 4.5
allowed-tools: Read, Grep
---

Read kb/todo/work.md and display its contents. Do not search the repo or use any other tools / agents.
```

I added it cause asking CC to list my todos was too slow (when I was lazy to just open the file in my editor). The skill sets Haiku which is a "dumb" model compared to Opus, so I don't recommend it for anything not simple. Now I run `/todos` and get results in ~5 seconds (still slow but much faster than before). I could also just added a line to my CLAUDE.md but preferred the skill to "enforce" the model and tools. 

Note: CC doesn't explicitly tell you which model it used for a skill, but I trust it and don't bother to switch to Haiku first with `/models` before I run it. Maybe you can tell with OpenTelemetry integration but I didn't bother to find out.

2. *The Profiling Skill*.

I use CC often to monitor logs (live runs or Datadog CSV exports - I didn't connect it to DD yet). Claude knows how to profile Python much better than me, but it still required a lot of back-and-forth, and some of the things it did were sub-optimal for what I was after, like exploring the codebase. I ended up with a fairly involved skill [profile-python](https://github.com/yuval/dotfiles/blob/main/claude/skills/profile-python/SKILL.md) to personalize the profiling style to the kind of systems that I'm currently working on - heavy async processing of a lot of data through multi-provider LLM pipelines. 

### plugins

Run `/plugins` to see what's installed or discover new ones. My current setup:

```
 Installed plugins (4):
  ● pyright-lsp
    Python language server (Pyright) for type checking and code intelligence
  ● frontend-design
    Frontend design skill for UI/UX implementation
    directly from source repositories into your LLM context.
  ● code-review
    Automated code review for pull requests using multiple specialized agents with confidence-based scoring
```

I like the code review plugin and find it useful for finding issues. But it eats a lot of tokens and can be slow. I've seen it consumes 150K tokens for small PRs cause it executes many parallel review sub-agents. 


## Context & Memory

Before we dive in, make sure you have the context percentage in your CC statusline. You can run the `/statusline` command to configure it:
```
❯ /statusline

⏺ statusline-setup(Configure statusline from PS1)
  ⎿  Done (7 tool uses · 13.0k tokens · 25s)

⏺ Your status line is already configured and looks good. It shows current directory, git branch with dirty indicator, model name, and context usage. No changes needed.
```

[This one](https://github.com/yuval/dotfiles/blob/main/claude/statusline.sh) is mine. For context management it's important to have the context percentage on the statusline (you can also run `/context` for a detailed breakdown).

### Memory Layers

Memory in CC has many layers (see [docs](https://docs.anthropic.com/en/docs/claude-code/memory)), and they even forgot to list user-level rules (~/.claude/rules/). 

For me, User Memory is the most important, followed by Project Memory. You can find mine [here]( https://github.com/yuval/dotfiles/blob/main/claude/CLAUDE.md). It links out to specific rules files ([python.md](https://github.com/yuval/dotfiles/blob/main/claude/rules/python.md), [testing.md](https://github.com/yuval/dotfiles/blob/main/claude/rules/testing.md), [database.md](https://github.com/yuval/dotfiles/blob/main/claude/rules/database.md), etc.). You can dump everything into CLAUDE.md, but I prefer separating them for organization. Run `/memory` to verify whatever you have is set up correctly.

**Auto-memory**. This is a new feature it seems. These are notes Claude writes for itself stored in `~/.claude/projects/<project>/memory/MEMORY.md`. According to the docs, the first ~200 lines of `MEMORY.md` are injected into every conversation's system prompt for that project. I'm not entirely sure why this needs to be separate from the project-level CLAUDE.md. I went over all my MEMORY.md files and most were empty.   

## Some Useful Commands & Workflow

- `/clear`: I use this often when previous context isn't useful anymore.
- `/compact`: Claude runs this automatically when you hit the context limit (based on `/context` ~33k tokens reserved), but you can run it manually with instructions as needed. E.g., `/compact keep only the profiling summary and initial task and drop intermediate outputs`. BTW, while compaction is a lossy operation, I noticed CC saves a file on disk with the full transcript and points to it in the summary
- `Expansions` (Ctrl-o / Ctrl-e): Ctrl-o expands tool calls. It's verbose, but it's how i often debug the agent. Sometimes Claude fails a command (like a permission error) and falls back to something else. You can only catch and fix that (e.g., by granting sudo access) if you check the logs. Ctrl-e shows the full history. 
- `Parallel Sessions`. Sessions are tied to directories and a common workflow is to run parallel Claude sessions using git worktrees. On the Desktop/Web apps, every new "chat" is a new session.
- `Bash Mode`. You can toggle bash mode with `!`. I switch between the two often. It supports ctrl-r for history search across both bash and even Claude history, which is cool.

## Codex

I primarily use CC but I do use Codex sometimes. Mostly recently, CC has been having more down times or just being slower than usual. I'm still in shock how these companies deal with all the demand and manage to serve so many tokens without being down much more. Kudos to them. Anyhow, competition is always good and I don't like to be without a coding agent for too long, so it's nice that switching agents is easy. I do plan to use Codex more often as it seems getting better. For now I only added an [AGENTS.md](https://github.com/yuval/dotfiles/blob/main/codex/AGENTS.md) similar to my user-level CLAUDE.md and a few [safety rules](https://github.com/yuval/dotfiles/blob/main/codex/rules/safety.rules) taken directly from my CC's deny list. I'll keep tweaking this as I go.