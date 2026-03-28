+++
title = "What I Learned Shipping a Visual Dashboard Builder in 3 Days"
slug = "what-i-learned-shipping-a-visual-dashboard-builder-in-3-days"
description = "Notes on using multiple LLMs to research, design, implement, and recover a UI-heavy feature under time pressure."
date = 2026-03-28T17:20:22+02:00
draft = false

[taxonomies]
tags = ["engineering", "ai"]

[extra]
toc = true
comment = false
+++

A customer wanted a visual dashboard builder (where you click buttons and create your own dashboard).
They needed this since yesterday™, but I absolutely didn't want to compromise on quality as I knew having this poorly implemented would bite us a lot in the future.

In 3 days we shipped a usable first version of a visual dashboard builder where users could:
- compose a query by selecting a source, dimensions, measures, filters, sorts, limit, and timezone
- use the result of that query to configure a widget, including chart settings and previewing the output
- place one or more widgets into a dashboard grid, then drag, drop, and resize them until the layout felt right
- save, edit, and delete dashboards once they were happy with the result

Under the hood, each widget stored structured query intent rather than relying on SQL as the editing format. That made the builder easier to reopen, validate, and evolve over time. The system could still compile that intent into executable ClickHouse SQL for preview and persistence, but the source of truth for editing stayed in a format the UI could understand directly.

I decided to organise the work in 4 phases:
- research planning
- architecture planning
- implementation planning
- implementation

During this work I learned that:
- In my experience on this project, Codex was amazing at research and following a plan but weak at UI work
- In my experience on this project, Claude Code did decent research, could follow a plan, and was much stronger at UI work
- As such, from now on I shall remember:
> Switching large language model is faster than forcing it to do something it isn't good at
- It is hard to know in advance which model will be best at a given task, so I use several and keep the strongest output
- User interfaces deserve their own spec (which can be a .html file)

Here's how I proceeded in more detail:

# Research plan

First, I asked an LLM to produce a research prompt:

```markdown
## Context
## Our tech stack
## What the dashboard builder needs to do
## What I need from this research
### 1. Architecture & design patterns
### 2. Clickhouse specific challenges
### 3. Phoenix LiveView specific considerations
### 4. Common bugs & failure modes
### 5. Query safety & guardrails
### 6. Dashboard layout & composition
### 7. Lessons from existing tools
### 8. Build vs. integrate
## Output format
Please structure your response with clear sections matching the numbered topics above. For each section:
- Lead with the **key insight or recommendation**
- Follow with **supporting evidence** (how other tools do it, common patterns)
- End with **specific pitfalls to avoid**
- Where relevant, suggest **concrete Elixir/LiveView patterns** or pseudocode
```

Then I asked the following LLM platforms to produce their own research outputs:
- claude code (cli)
- codex (cli)
- claude (web)
- gemini (web)
- perplexity (web)

After that, I asked Claude Code which research output was the best and to my surprise it said Codex!
I then asked it to add to the Codex research output any insights from other research outputs.

Once we had compiled this information, we moved on to the next step -- architecting the solution.

# Architecture plan

The purpose of this phase is to commit to a concrete architecture/design and set of features based on the knowledge gathered from the previous phase.
I asked Claude Code to create an architecture plan and then asked it to review it while brainstorming with Codex (you can set up the Codex MCP in your `.mcp.json` so Claude can use Codex).
This created me a sensible design plan that looks roughly like so:

```markdown
# <Feature> Architecture Plan

## Goal
## Context / Problem
## Target Architecture
## User Flow
## Storage Model

## Safety Constraints
- metadata / validation context
- permissions / limits / performance guardrails / failure handling

## Scope
```

# Implementation plan

The purpose of this phase is to commit to a specific implementation of the architecture we've specified above.
Here again, I used the same process as before and asked Claude Code to draft an implementation plan and then asked it to review it with Codex.
The implementation plan was defined as a series of steps but the plan was so huge that I ended up cutting it into multiple phases where each phase was its own plan:
- foundation with query compilation
- routing seam
- dashboard builder UI
- save, edit and delete flow

```markdown
# <Feature> Implementation Plan

## Goal

## Execution Policy
- test-first / commit cadence / validation expectations

## Safety Guardrails
- invariants that must never be broken

## Completion Gate
- format / compile / lint / tests / QA required before done

## File Plan
### New files
### Modified files
### Test files

## Tasks
### Task 1: <name>
- Purpose
- Files
- Required outcomes
- Test outlines
- Manual spot checks
- Step-by-step checklist

### Task 2: <name>
- same shape

```

# Implementation

For the implementation phase I told Codex to:
- create a git worktree for the implementation phase
- follow the plan
- create a PR, assign it to me
- move on to the next phase of the plan

Implementation was not a straight line. The backend-heavy parts responded well to planning and agent execution, but the UI exposed a gap in the spec. The first versions were technically functional but hard to understand and visually inconsistent with the rest of the app. At that point I stopped trying to prompt my way out of an underspecified UI problem and wrote down a proper design direction instead: first a design-system HTML file based on the existing app, then low-fidelity wireframes, then a higher-fidelity wireframe, and only then another implementation pass.

So I changed the process:
- asked both Claude Code and Codex to produce a design system in a self-contained .html file out of what we already had in the application.
  Claude Code was much better than Codex here so this is the design system I selected.
- asked Claude via Claude.ai to implement a low fidelity wireframe (that we iterated over multiple times)
- asked Claude via Claude.ai to implement a high fidelity wireframe using the previously created design system

I used Claude via Claude.ai instead of via Claude Code as I didn't want Claude to be distracted by existing code and practices, etc.

# Conclusion

Three things I'd do again:
- use multiple models early for research and planning, then keep the best output instead of forcing one model to do everything
- treat visual builders as two separate problems: query architecture and UI specification
- store structured authoring state for visual features, and only compile to SQL at the execution boundary

That combination let me move fast without turning the feature into a future maintenance problem. I would never have gotten the same result in the pre-agentic coding era.

The next thing I want to build is a more reactive development system around this workflow: when I move a PR from draft to ready for review, an LLM reviews it automatically; when CI goes red, an agent starts trying to make it green; when a design changes, downstream implementation work updates with it.

That kind of system creates more parallel work so I can spend more time on product, architecture, and judgment instead of manually pushing every step forward.
