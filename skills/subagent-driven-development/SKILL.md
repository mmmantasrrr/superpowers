---
name: subagent-driven-development
description: Use when executing implementation plans with independent tasks in the current session
---

# Subagent-Driven Development

Execute plan by dispatching tasks to the local `dispatch-harness`, with two-stage review after each: spec compliance review first, then code quality review.

**Core principle:** Fresh isolated task (via dispatch) + two-stage review (spec then quality) = high quality, fast iteration

## When to Use

```dot
digraph when_to_use {
    "Have implementation plan?" [shape=diamond];
    "Tasks mostly independent?" [shape=diamond];
    "Stay in this session?" [shape=diamond];
    "subagent-driven-development" [shape=box];
    "executing-plans" [shape=box];
    "Manual execution or brainstorm first" [shape=box];

    "Have implementation plan?" -> "Tasks mostly independent?" [label="yes"];
    "Have implementation plan?" -> "Manual execution or brainstorm first" [label="no"];
    "Tasks mostly independent?" -> "Stay in this session?" [label="yes"];
    "Tasks mostly independent?" -> "Manual execution or brainstorm first" [label="no - tightly coupled"];
    "Stay in this session?" -> "subagent-driven-development" [label="yes"];
    "Stay in this session?" -> "executing-plans" [label="no - parallel session"];
}
```

**vs. Executing Plans (parallel session):**
- Same session (no context switch)
- Fresh isolated session per task (no context pollution)
- Two-stage review after each task: spec compliance first, then code quality
- Faster iteration (no human-in-loop between tasks)

## The Process

```dot
digraph process {
    rankdir=TB;

    subgraph cluster_per_task {
        label="Per Task";
        "Run `dispatch run` for implementer (./implementer-prompt.md)" [shape=box];
        "Implementer (via dispatch) asks questions?" [shape=diamond];
        "Answer questions, provide context" [shape=box];
        "Implementer completes task, tests, commits" [shape=box];
        "Run `dispatch run` for spec reviewer (./spec-reviewer-prompt.md)" [shape=box];
        "Spec reviewer confirms code matches spec?" [shape=diamond];
        "Implementer (via dispatch) fixes spec gaps" [shape=box];
        "Run `dispatch run` for code quality reviewer (./code-quality-reviewer-prompt.md)" [shape=box];
        "Code quality reviewer approves?" [shape=diamond];
        "Implementer (via dispatch) fixes quality issues" [shape=box];
        "Mark task complete in TodoWrite" [shape=box];
    }

    "Read plan, extract all tasks with full text, note context, create TodoWrite" [shape=box];
    "More tasks remain?" [shape=diamond];
    "Run `dispatch run` for final code reviewer" [shape=box];
    "Use superpowers:finishing-a-development-branch" [shape=box style=filled fillcolor=lightgreen];

    "Read plan, extract all tasks with full text, note context, create TodoWrite" -> "Run `dispatch run` for implementer (./implementer-prompt.md)";
    "Run `dispatch run` for implementer (./implementer-prompt.md)" -> "Implementer (via dispatch) asks questions?";
    "Implementer (via dispatch) asks questions?" -> "Answer questions, provide context" [label="yes"];
    "Answer questions, provide context" -> "Run `dispatch run` for implementer (./implementer-prompt.md)";
    "Implementer (via dispatch) asks questions?" -> "Implementer completes task, tests, commits" [label="no"];
    "Implementer completes task, tests, commits" -> "Run `dispatch run` for spec reviewer (./spec-reviewer-prompt.md)";
    "Run `dispatch run` for spec reviewer (./spec-reviewer-prompt.md)" -> "Spec reviewer confirms code matches spec?";
    "Spec reviewer confirms code matches spec?" -> "Implementer (via dispatch) fixes spec gaps" [label="no"];
    "Implementer (via dispatch) fixes spec gaps" -> "Run `dispatch run` for spec reviewer (./spec-reviewer-prompt.md)" [label="re-review"];
    "Spec reviewer confirms code matches spec?" -> "Run `dispatch run` for code quality reviewer (./code-quality-reviewer-prompt.md)" [label="yes"];
    "Run `dispatch run` for code quality reviewer (./code-quality-reviewer-prompt.md)" -> "Code quality reviewer approves?";
    "Code quality reviewer approves?" -> "Implementer (via dispatch) fixes quality issues" [label="no"];
    "Implementer (via dispatch) fixes quality issues" -> "Run `dispatch run` for code quality reviewer (./code-quality-reviewer-prompt.md)" [label="re-review"];
    "Code quality reviewer approves?" -> "Mark task complete in TodoWrite" [label="yes"];
    "Mark task complete in TodoWrite" -> "More tasks remain?";
    "More tasks remain?" -> "Run `dispatch run` for implementer (./implementer-prompt.md)" [label="yes"];
    "More tasks remain?" -> "Run `dispatch run` for final code reviewer" [label="no"];
    "Run `dispatch run` for final code reviewer" -> "Use superpowers:finishing-a-development-branch";
}
```

## Dispatch Harness Integration

This skill uses the local `dispatch-harness` to execute tasks in isolated background sessions.

**To run a task:**
Use the `run_shell_command` tool to execute:
`dispatch run --worker gemini --task-id <id> --workdir . "<prompt>"`

The `<prompt>` should combine the role's template (e.g., `./implementer-prompt.md`) with the specific task text from the plan.

## Prompt Templates

These templates should be read and included in the dispatch prompt:
- `./implementer-prompt.md` - Role: Implementer
- `./spec-reviewer-prompt.md` - Role: Spec Compliance Reviewer
- `./code-quality-reviewer-prompt.md` - Role: Code Quality Reviewer

## Example Workflow

```
You: I'm using Subagent-Driven Development to execute this plan.

[Read plan file once: docs/plans/feature-plan.md]
[Extract all tasks and create TodoWrite]

Task 1: Hook installation script

[Read ./implementer-prompt.md]
[Run shell command: dispatch run --worker gemini --task-id 1 --workdir . "Role: [implementer-prompt] \n Task: Hook installation script..."]

Implementer (via dispatch): "Done. Implemented, tested, and committed."

[Read ./spec-reviewer-prompt.md]
[Run shell command: dispatch run --worker gemini --task-id 1-spec --workdir . "Role: [spec-reviewer-prompt] \n Task: Review Hook installation script..."]

Spec reviewer: ✅ Spec compliant.

... [Continue for code quality review and next tasks]
```

## Advantages

**vs. Manual execution:**
- Dispatch sessions follow TDD naturally
- Fresh context per task (no confusion)
- Parallel-safe (isolated workdirs)
- Worker can ask questions (before AND during work)

**Quality gates:**
- Two-stage review: spec compliance, then code quality
- Review loops ensure fixes actually work
- Spec compliance prevents over/under-building
- Code quality ensures implementation is well-built

## Red Flags

**Never:**
- Start implementation on main/master branch without explicit user consent
- Skip reviews (spec compliance OR code quality)
- Proceed with unfixed issues
- Accept "close enough" on spec compliance (spec reviewer found issues = not done)
- Move to next task while either review has open issues

**If reviewer finds issues:**
- Implementer (via dispatch) fixes them
- Reviewer reviews again
- Repeat until approved
- Don't skip the re-review

## Integration

**Required workflow skills:**
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
- **superpowers:writing-plans** - Creates the plan this skill executes
- **superpowers:requesting-code-review** - Code review template for reviewer workers
- **superpowers:finishing-a-development-branch** - Complete development after all tasks

**Workers should use:**
- **superpowers:test-driven-development** - Workers follow TDD for each task
