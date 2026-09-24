# Workflows

A focused library of reusable workflows, operating guides, and agent playbooks.

Each top-level directory describes how to approach a specific technical domain or recurring delivery pattern. These guides are intended to give people, coding agents, and MCP-enabled tools a shared starting point: what the goal is, which boundaries matter, how work should be verified, and when a human decision is required.

## Current Workflows

| Directory | Purpose |
|---|---|
| [atomic](./atomic/) | AtomicQMS development, testing, environment, approval, and deployment guidance |
| [pythonqt](./pythonqt/) | Python/Qt development workflow guidance |

## Adding a Workflow

Create one self-contained top-level directory per workflow area. A good workflow guide should include:

- the intended outcome and non-goals
- required context and prerequisites
- a safe, repeatable step-by-step loop
- validation and stop conditions
- security, approval, or deployment boundaries
- links to the authoritative source repositories and code locations

Keep workflow guidance practical, specific, and current. Prefer a small durable guide over broad, duplicated documentation.

## Scope

This repository holds process and development guidance only. Application source code, production configuration, operational secrets, and project-specific implementation artifacts belong in their respective repositories.
