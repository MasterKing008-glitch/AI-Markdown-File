# AGENT.md

## Purpose

This file defines how an AI agent should execute tasks within this repository.

It focuses on workflow, decision-making, and safe execution within a Turborepo monorepo using RPC-based architecture.

The agent must follow this process before making changes.

---

## Execution Workflow

### 1. Understand the Task

- Read the request carefully
- Identify:
  - Scope (single file, service, or multiple workspaces)
  - Affected layers (API, web, database, RPC, providers)
- If unclear, ask clarifying questions before proceeding

---

### 2. Plan Before Implementation

For any non-trivial task, the agent must:

- Describe the approach
- List affected files and packages
- Identify risks (schema change, shared contract, cross-workspace impact)

Do not proceed with large changes without confirmation.

---

### 3. Scope Control

- Prefer minimal, focused changes
- Avoid modifying multiple apps or packages unless required
- Do not refactor unrelated code
- Do not introduce new abstractions unless necessary

---

### 4. Follow Architecture Rules

- Respect Turborepo structure:
  - apps/api
  - apps/web
  - packages/*

- Use RPC for internal communication (Cloudflare network)
- Do not introduce internal REST calls where RPC is expected
- Use shared contracts instead of duplicating types

---

### 5. Implementation Guidelines

- Keep route/RPC handlers thin
- Place logic in services
- Use Drizzle for database access
- Validate inputs using Zod
- Normalize external data (Facebook, future providers)

---

### 6. Working with Providers

- Use the provider abstraction (SocialProvider interface)
- Do not couple logic directly to Facebook
- Ensure new providers follow the same contract
- Normalize all provider data into internal DTOs

---

### 7. Database Safety

- Do not modify schema without approval
- Do not edit migration files directly
- Ensure migrations are generated properly
- Avoid destructive changes

---

### 8. Cross-Workspace Changes

The agent must ask before:

- Editing shared packages
- Changing RPC contracts
- Modifying turbo.json
- Adding dependencies
- Performing large refactors

---

### 9. Testing & Validation

Before completing a task:

- Ensure code compiles (TypeScript strict)
- Ensure lint passes
- Add tests where appropriate
- Avoid real external API calls in tests

---

### 10. Output Requirements

- Provide clear, minimal changes
- Avoid unnecessary explanations
- Highlight:
  - What changed
  - Why it changed
  - Any risks or follow-ups

---

## Safety Rules

- Never introduce secrets or sensitive data
- Never bypass validation
- Never break existing contracts without warning
- Never assume architecture changes are acceptable

---

## When to Ask for Clarification

The agent must pause and ask if:

- Requirements are unclear
- Multiple architectural approaches are possible
- Changes affect multiple systems
- A decision impacts long-term structure

---

## Final Rule

The agent must prioritize:

- Safety over speed
- Clarity over cleverness
- Consistency over innovation

If uncertain, ask before acting.