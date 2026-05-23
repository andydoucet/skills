---
name: code-structure
description: Use when multiple workflows duplicate the same operational logic, when deciding what belongs in actions vs shared services, or when refactoring repeated operational blocks across domain flows. Use when adding new features that share mechanics with existing ones.
---

# Service Layer Architecture

## Overview

Use a two-layer separation: actions orchestrate domain rules, while a service layer centralizes reusable operational mechanics.

This prevents duplicated code, inconsistent behavior, and bugs fixed in one path but not others.

## When to Use

- Multiple callers need the same low-level operation, such as sandbox creation, email sending, or payment processing.
- Operational logic is being copy-pasted between action files.
- A bug fix in one workflow does not propagate to other workflows doing the same thing.
- A new feature shares mechanics with existing flows.

Do not use this pattern when logic is truly domain-specific and used by only one caller.

## Core Pattern

```text
Orchestration Layer (Actions)          Service Layer (Shared Mechanics)
|- owns business rules                 |- owns reusable operations
|- owns state transitions              |- owns provider/SDK interactions
|- owns auth/ownership checks          |- owns command execution details
|- owns failure classification         |- owns health checks / readiness
|- owns retries / user-facing errors   `- returns structured results
`- calls service functions
```

Rule of thumb:

- "What this product flow means" belongs in actions.
- "How to do this operation reliably" belongs in the service layer.

## Quick Reference

| Design Principle | Do | Don't |
|---|---|---|
| API shape | Use composable capability blocks | Create one giant "do everything" method |
| Inputs/outputs | Use explicit params and structured returns | Depend on hidden global state or reach into the DB |
| Migration | Extract one block, replace one caller, verify, then migrate the rest | Refactor everything at once |
| Domain logic | Keep auth, policy, and error classification in actions | Let services mutate domain state directly |
| Extraction trigger | Extract logic repeated across 2+ callers | Extract logic used once |

## Designing Service Functions

Design service functions as capability blocks, not monoliths:

```ts
createManagedSandbox(...)
prepareRepo(...)
detectPackageManager(...)
installDependencies(...)
runBuildCommand(...)
startSandboxRuntime(...)
```

Each function should:

- Accept all required data as explicit parameters.
- Return structured outputs, such as `{ ready, previewUrl, proxyPort }`.
- Avoid direct database or domain-state access.
- Make failure explicit through structured results or thrown errors, not swallowed errors.

This lets callers choose strict or relaxed behavior per flow.

## Migration Checklist

When extracting shared logic:

1. Write the flow in action code first so behavior is clear.
2. Mark repeated operational chunks across callers.
3. Extract only repeated, non-domain chunks to a service.
4. Replace one caller, verify it, then replace remaining callers.
5. Keep domain policy in actions, including auth, status transitions, and error classification.
6. Run verification: typecheck, lint, and confirm all affected flows still work.

## Anti-Patterns

| Anti-Pattern | Problem |
|---|---|
| God service | One huge function hides all control flow |
| Leaky service | Service mutates database tables directly |
| Inconsistent API | Functions use different argument styles and error semantics |
| Over-abstraction | Logic used by only one caller gets extracted anyway |

## Example: Email Service

```ts
// emailService.ts - shared mechanics
export async function sendWelcomeEmail(params: { to: string; name: string }) {
  const html = `<h1>Welcome ${params.name}</h1>`;
  await emailProvider.send(params.to, "Welcome", html);
}

// userSignup.ts - orchestration owns when to send
if (user.marketingOptIn) {
  await sendWelcomeEmail({ to: user.email, name: user.name });
}

// adminInvite.ts - different business rule, same mechanic
await sendWelcomeEmail({ to: invitee.email, name: invitee.name });
```

## Mental Model

```text
New feature? -> Write in action first -> See repeated ops? -> Extract to service
                                      -> No repetition?  -> Keep in action
```

Architecture in one sentence: actions orchestrate domain rules, while the service layer centralizes reusable operational mechanics with a composable, explicit-input API.
