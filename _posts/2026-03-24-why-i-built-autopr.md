---
layout: post
title: "Why I Built AutoPR"
subtitle: "From ticket to PR without babysitting a terminal"
date:   2026-03-24 08:00:00 +0100
categories: general ai developer-tools automation
---

Lately I wanted to spend more time working with AI-assisted code generation. Not in the abstract, but in the real day-to-day workflow where a ticket becomes a proposal, then a concrete implementation, and eventually a pull request.

The problem was not that the models were incapable. The problem was the workflow around them.

Most of the tools I tried felt like a single long conversation attached to a terminal window. You start a task, the agent begins working, and then you wait. And wait. The terminal stays open, the context is buried somewhere in scrollback, and the whole thing feels strangely synchronous for a workflow that should really be staged and asynchronous.

That annoyed me more than the occasional bad code generation.

What I actually wanted was something much closer to how I already think about software work:

1. Gather the relevant context from the ticket.
2. Include parent ticket or epic information if it exists.
3. Let the system propose a solution.
4. Review that proposal and iterate on it.
5. Only then move on to implementation.
6. End with a pull request that is ready to review.

And all of that should happen in the background.

I did not want to sit in front of an open terminal just to keep an eye on a model. I wanted a small dashboard where I could see which tickets are still being investigated, which ones are waiting for human input, which ones are already in implementation, and which ones are ready for review.

That is why I started building **AutoPR**.

The project is available here: [github.com/Neokil/AutoPR](https://github.com/Neokil/AutoPR).

# The core idea
AutoPR is a local-first workflow tool that turns a ticket into a staged AI pipeline.

It has a CLI, a background server, and a web UI. The important part is that the CLI and the UI support the same core workflow.

I started with the CLI because it was simpler to build and easier to iterate on. Once the workflow itself felt right, I moved on to the web UI, because it is simply much easier to use for day-to-day work and it gives me the dashboard view I wanted across multiple repositories.

Instead of treating AI work as one opaque run, AutoPR breaks it into explicit workflow states:

- `queued`
- `investigating`
- `proposal_ready`
- `waiting_for_human`
- `implementing`
- `validating`
- `pr_ready`
- `done`
- `failed`

That structure matters.

The first stage fetches ticket details, and not just the bare minimum. The workflow is designed to pull in related context as well, including parent tickets and epics when available. In my case that matters because the ticket itself often does not contain the whole story. The actual reasoning is spread across multiple layers of project management context.

Once that context is available, the system generates a proposal. That proposal is not the final output. It is intentionally a review point.

This was one of the main things I felt was missing in many AI coding workflows. I do not want to jump directly from "here is a vague ticket" to "here is a branch full of generated code." I want a step in between where the solution can be challenged, corrected, and sharpened before implementation starts.

So AutoPR makes that explicit: investigate first, pause for review, then continue.

# Why configurability mattered from the start
Another important design goal was flexibility.

I did not want this tool to be hard-wired to one model vendor or one AI coding assistant. The point of AutoPR is not to enforce a specific LLM. The point is to provide the workflow and structure around it.

So the LLM integration is intentionally configurable. In the config file, you can define which tool should be used and how its CLI needs to be called. That means the workflow layer stays stable even if the underlying model or CLI changes.

The prompts are configurable too. AutoPR stores them in the home directory so they can be edited freely. That was important to me because prompt iteration is part of the product. I did not want prompt behavior hidden inside the binary or buried in source code. I wanted it to be easy to tweak how ticket fetching, investigation, implementation, or PR generation work without rebuilding the tool.

# Why the async part matters
The async aspect is not just a convenience feature. For me it is the entire point.

If I need to keep a terminal open and watch a long-running agent session, the tool is still demanding synchronous attention from me. It may automate some typing, but it is not really fitting into how I want to work.

I want to queue a few tickets, switch back to something else, and come back later to see:

- which tickets need my feedback,
- which ones have completed implementation,
- which ones failed validation,
- and which ones already have a PR draft ready.

That changes the interaction model completely.

Instead of having a one-to-one relationship with an agent session, I get a small work queue. The AI becomes something closer to a background worker. I review outcomes at the right checkpoints rather than supervising every token it produces.

That feels much more natural.

# What AutoPR does today
The current version already reflects most of that idea.

It can run ticket workflows in the background, persist state on disk, and show the current status in a web UI. It stores ticket artifacts per repository, including the fetched ticket data, logs, the proposal, the final solution notes, and the generated PR description.

The prompts are also separated by stage:

- one prompt for fetching ticket context,
- one for investigation,
- one for implementation,
- and one for PR generation.

That separation is important because the questions I want to ask in each phase are different.

When investigating, I want problem framing, likely files to change, risks, and open questions.

When implementing, I want code changes, validation, and a clear summary of what was done.

When generating the PR, I want something that reads like a reviewable change description rather than another chain-of-thought dump.

Another detail I care a lot about is how ticket context is fetched.

Rather than writing direct integrations for every ticketing system myself, I am using the LLM with a preconfigured MCP setup to retrieve ticket information. That means the workflow can ask the model to fetch the ticket, parent ticket, and epic context through the tools it already has access to.

This is useful for two reasons.

First, it keeps AutoPR itself much simpler. I do not need to build and maintain one custom integration after another inside the project.

Second, it makes the whole thing more flexible. If I want to use a different ticket system, I can configure the LLM with the right MCP and adjust the query a bit instead of rewriting the application architecture around that system.

The UI mirrors the staged workflow. Tickets appear with their current status, active jobs, and repository context. A ticket can wait for approval, be resumed with feedback, move on to implementation, and eventually land in a PR-ready state.

In other words: the process is visible.

That visibility is another thing I was missing from terminal-based agent workflows. Once everything lives in scrollback, the system becomes harder to trust. A dashboard is not just prettier. It makes the workflow inspectable.

# Why I find this more useful than a single agent loop
I do not think the best AI coding workflows are the ones where the model gets maximum autonomy as early as possible.

In practice, the expensive mistakes usually happen before implementation even starts:

- the ticket was misunderstood,
- the epic context was ignored,
- the proposed direction was too broad,
- or the model started changing the wrong files for the wrong reason.

If you only review at the very end, you often discover that the implementation is wrong in a structural way. At that point, you are not reviewing code anymore. You are undoing wasted work.

A staged workflow is slower at the beginning, but faster where it matters. It gives you a controlled checkpoint after the investigation phase and before code generation becomes expensive.

That is the main design principle behind AutoPR.

# What I want next
There is still plenty to improve.

Right now I already have the basic loop I wanted: fetch context, propose a solution, get human input, implement, validate, and prepare a PR. But the broader idea is to make that flow feel even more like a small asynchronous engineering system and less like a wrapper around one AI command.

The interesting part is not only better prompts or better model output.

The interesting part is workflow design.

How do we decide which stages require human approval? How do we show confidence or risk? How do we feed review comments back into the loop cleanly? How do we make it obvious which tickets are worth opening right now because they are actually ready for review?

Those questions feel much more valuable to me than simply asking whether the next model is five percent better at writing code.

# Closing thoughts
AutoPR started from a pretty simple frustration: I was annoyed that AI coding tools often forced me into one long, synchronous terminal session.

What I wanted instead was staged work in the background, with a clear dashboard and clear review points.

So I started building exactly that.

It is still early, but even in its current form it already feels closer to the way I want to collaborate with AI on software work: not as a terminal performance I have to watch live, but as an asynchronous system that turns tickets into reviewable results.
