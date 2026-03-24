# AGENTS.md

## Purpose

This document provides operating guidelines for autonomous coding agents working in this repository.

## Repository Snapshot

- Project is currently minimal (`README.md` exists with basic content).
- Keep changes small, explicit, and easy to review.

## Agent Workflow

1. Understand the user request and inspect relevant files before editing.
2. Prefer minimal diffs that solve the requested task directly.
3. Validate changes locally when feasible.
4. Commit with clear messages describing intent and scope.
5. Push branch updates and keep PR description aligned with actual changes.

## Editing Rules

- Do not rewrite unrelated files.
- Preserve existing file formatting and style where possible.
- Add comments only when they improve clarity for non-obvious logic.
- Avoid adding unnecessary dependencies.
- If adding dependencies is required, use the project package manager and latest stable version.

## Verification

- Run the smallest relevant checks first (lint/typecheck/unit tests).
- If no automated checks exist, perform a reasonable manual sanity review.
- Report what was verified and any residual risks.

## Git & PR Practices

- Work on the current feature branch unless instructed otherwise.
- Use focused commits (one logical change per commit).
- Do not force-push unless explicitly requested.
- Keep PRs as draft by default unless the user asks for ready-for-review.

## Communication

- Be concise, direct, and transparent about what changed.
- Surface blockers early with concrete options.
- State assumptions explicitly when requirements are ambiguous.

