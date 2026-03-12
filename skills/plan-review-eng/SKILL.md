---
name: plan-review-eng
description: Use when reviewing a technical design for architectural soundness during ticket planning, or for code quality, test strategy, and performance review during ticket execution
---

# Engineering Plan Review

## Overview

Technical review of a design or implementation. Two modes depending on where you are in the workflow:

- **Planning mode** (during ticket creation): Architecture review only — component boundaries, data flow, failure modes
- **Execution mode** (during ticket implementation): Code quality, test strategy, and performance review

**Announce at start:** "Running engineering review in [planning/execution] mode."

## Planning Mode — Architecture Review

Use during `creating-linear-tickets`, after CEO review has locked scope.

### What to review (3-5 questions)

1. **What already exists?** — What existing code, components, or patterns partially solve this? Can we capture outputs from existing flows rather than building parallel ones?
2. **Component boundaries** — Where do the new pieces fit in the existing architecture? What are the interfaces? Draw an ASCII diagram of the data flow.
3. **Complexity check** — If the design touches >8 files or introduces >2 new abstractions, challenge whether the same goal can be achieved with fewer moving parts. Minimal diff wins.
4. **Failure modes** — For each new codepath or integration point, describe one realistic production failure scenario. How would the user experience it? Is it silent or visible?
5. **Dependency analysis** — What needs to exist before this can be built? What's the natural ordering of work? This informs ticket sequencing.

### Output

- ASCII architecture diagram showing data flow
- File list: what's created, what's modified
- Failure modes with visibility assessment
- Dependency graph for ticket ordering
- "What already exists" section — existing code that can be reused

### Interaction Rules

- One question at a time, opinionated recommendations
- Lead with recommendation: "I'd structure it as X because..."
- Present 2-3 options with lettered choices where genuine tradeoffs exist
- For each option: one line on effort, risk, and maintenance burden

## Execution Mode — Code Quality, Tests, Performance

Use during `starting-linear-ticket` Step 4 (brainstorming), when you're in the worktree looking at actual code.

### Code Quality Review (3-5 questions)

- **Minimal diff** — Can we achieve this with fewer files/abstractions? Are we over-engineering?
- **DRY violations** — Does this duplicate existing patterns? Check for similar code elsewhere.
- **Edge cases** — What inputs, states, or timing conditions break this? List them explicitly.
- **Error handling** — Are errors visible or silent? Does the user see a clear message or a blank screen?

### Test Strategy Review (3-5 questions)

- **Codepath diagram** — Diagram all new codepaths and branching outcomes (ASCII)
- **Coverage mapping** — For each codepath: what test covers it? (unit, integration, e2e, manual)
- **Negative paths** — What should NOT happen? Are there tests for those?
- **Acceptance criteria verification** — Does the test plan match the ticket's acceptance criteria?

### Performance Review (2-3 questions)

- **Query patterns** — N+1 queries? Unnecessary Supabase round-trips? Can anything be batched?
- **Bundle impact** — New dependencies? New client-side code that affects load time?
- **Caching** — Any data that's fetched repeatedly but rarely changes?

### Output

- Updated test plan with codepath diagram
- List of edge cases to handle
- Performance concerns (if any)
- Recommendations prioritized: must-fix vs nice-to-have

## Common Mistakes

### Running full review during planning
- **Problem:** Code quality and test strategy are hypothetical without actual code
- **Fix:** Architecture only during planning. Save b-d for execution when you can read the code.

### Skipping "what already exists"
- **Problem:** Building new when you could extend existing
- **Fix:** Always search for existing patterns first. grep before you create.

### Vague failure modes
- **Problem:** "It could fail" without specifics
- **Fix:** Name the exact failure: "Supabase RLS denies access if user_id isn't set, returning empty array instead of error"

## Red Flags
- "The architecture is obvious, skip to implementation"
- "We'll figure out edge cases during coding"
- "Tests can come after"
- "Performance isn't a concern at this scale"

**All of these mean: Run the review. These are exactly the moments it catches something.**
