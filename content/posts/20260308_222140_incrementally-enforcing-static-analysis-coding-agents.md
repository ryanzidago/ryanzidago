+++
title = "Incrementally Enforcing Static Analysis with Coding Agents"
slug = "incrementally-enforcing-static-analysis-coding-agents"
description = "A practical strategy for adding linting and static analysis to a legacy codebase using coding agents."
date = 2026-03-08T00:00:02
draft = false

[taxonomies]
tags = ["engineering", "ai"]

[extra]
toc = true
comment = false
+++

AI agents are becoming part of the development workflow, and they're only as good as the feedback your codebase gives them.

OpenAI calls this [harness engineering](https://openai.com/index/harness-engineering/): the idea that you should wrap your codebase in linters, structural tests, and automated checks that act as guardrails for both human developers and AI agents.

> In practice, we enforce these rules with custom linters and structural tests, plus a small set of "taste invariants." [...] we statically enforce structured logging, naming conventions for schemas and types, file size limits, and platform-specific reliability requirements with custom lints.

Great. But what if your codebase doesn't have any of this?

## The problem

In a new project, it's fairly easy to add new linter rules or CI checks as there are very few existing offenses against those rules.

In a large codebase that has seen many developers over the years, this is another story.

You may have never used types or lints. You add them; now you get hundreds and hundreds of errors, lints, warnings, refactoring opportunities etc.

How do you best proceed then?
Do you let the AI fix everything, everywhere, all at once?
You can end up with PRs with +10_000 lines of changes ... This is too much to review, so the probability that bugs slip through is higher. It's also a merge conflict magnet against any parallel work, and if something does break, rolling back means losing all the fixes; not just the problematic one.

## The strategy

Here's what I suggest:

- add the tool
- enable it in the CI
- now locally, run the tool to see which config/rules produce offenses
- disable any rules that produce offenses and merge your PR into main

This will prevent contributors from producing more offenses and set the best possible baseline.

Now that you have a best case, let's look at how to bring the rest of the rules into play. Ask your favorite coding agent:

> Here's the list of rules that we want to enable in this project. Create stacked PRs where each PR enables (and eventually fix all offenses related to) one rule. Once all the tests and precommit checks and the CI passes, create the PR and move on to the next rule; rinse and repeat until all the following rules are enabled.

A word of caution: not all rules are equal. Fixing import ordering is mechanical; adding types to a legacy module with implicit contracts can surface deeper design issues. If a rule's fixes start snowballing, it's fine to skip it and come back later. The point is steady progress.

Here's what the progression looks like:

| Rule                 | PR #0 | PR #1 | PR #2 | PR #3 | PR #4 | PR #5 | PR #6 |
| -------------------- | ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| `eqeqeq`             | ✓     | ✓     | ✓     | ✓     | ✓     | ✓     | ✓     |
| `no-eval`            | ✓     | ✓     | ✓     | ✓     | ✓     | ✓     | ✓     |
| `import/order`       | ✕     | **✓** | ✓     | ✓     | ✓     | ✓     | ✓     |
| `no-console`         | ✕     | ✕     | **✓** | ✓     | ✓     | ✓     | ✓     |
| `no-unused-vars`     | ✕     | ✕     | ✕     | **✓** | ✓     | ✓     | ✓     |
| `naming-convention`  | ✕     | ✕     | ✕     | ✕     | **✓** | ✓     | ✓     |
| `no-explicit-any`    | ✕     | ✕     | ✕     | ✕     | ✕     | **✓** | ✓     |
| `strict-null-checks` | ✕     | ✕     | ✕     | ✕     | ✕     | ✕     | **✓** |

**PR #0**: Add tool to CI. Enable `eqeqeq` and `no-eval` (zero offenses). Disable the rest.
**PR #1–#6**: One rule per PR. Fix all offenses. Merge. Next.

There are various ways to slice this incremental work:

- per rule
- per rule per area (backend, frontend)
- per rule per domain (accounts, invoices, orders)
- per file

Find the granularity where you are comfortable reviewing and releasing changes.

The investment pays off twice: a cleaner codebase for your team and better guardrails for AI-assisted development.
