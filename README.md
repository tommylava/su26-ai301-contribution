# Contribution 1: Fix async-await-in-loop Performance Issue

**Contribution Number:** 1
**Student:** Thomas Lavadinho
**Issue:** [wso2/product-is#27874](https://github.com/wso2/product-is/issues/27874)
**Status:** Phase II Complete

## Why I Chose This Issue

I switched to this issue from a Session Desktop one because this one is way more
approachable. The React Doctor tool already flagged exactly which files and line numbers
have the problem, so I didn't have to go hunting through a huge codebase trying to
figure out what was wrong. The fix pattern is also clear — it's the same change in
14 places. And it's a real performance improvement, not just a style thing, which made
it feel worth doing.

## Understanding the Issue

**Problem Description:**
In 14 files across the identity-apps repo, there are `await` calls sitting inside
`for...of` loops. This means every async API call has to fully finish before the next
one even starts. If all the calls are independent (which they are here), that's just
wasted time.

**Expected Behavior:**
Independent async calls should all fire at the same time using `Promise.all`, so the
total wait time is just however long the slowest single call takes instead of adding
them all up.

**Current Behavior:**
Every call waits for the one before it. If each call takes 200ms and there are 5 of
them, you're waiting 1000ms instead of ~200ms.

**Affected Components:**
14 files in the `features/` directory of `wso2/identity-apps`:
- `features/admin.administrators.v1/wizard/add-administrator-wizard.tsx:279`
- `features/admin.connections.v1/components/edit/settings/authenticator-settings.tsx:442`
- `features/admin.connections.v1/components/edit/settings/outbound-provisioning-settings.tsx:239`
- `features/admin.console-settings.v1/hooks/use-enterprise-login-config.ts:230`
- `features/admin.console-settings.v1/hooks/use-enterprise-login-config.ts:547`
- `features/admin.copilot.v1/api/copilot-api.ts:233`
- `features/admin.copilot.v1/store/actions/copilot.ts:526`
- `features/admin.flow-builder-core.v1/api/use-resolve-custom-text-preference.ts:140`
- `features/admin.flow-builder-core.v1/plugins/plugin-registry.ts:180`
- `features/admin.server-configurations.v1/pages/connector-listing-page.tsx:284`
- `features/admin.template-core.v1/hooks/use-initialize-handlers.ts:91`
- `features/admin.template-core.v1/hooks/use-initialize-handlers.ts:114`
- `features/admin.template-core.v1/hooks/use-submission-handlers.ts:118`
- `features/admin.template-core.v1/hooks/use-validation-handlers.ts:114`

## Reproduction Process

### Environment Setup

I forked `wso2/identity-apps` to `tommylava/identity-apps` and cloned it locally.
First push attempt failed with a 403 because I had originally cloned the original repo
directly instead of my fork — fixed that by updating the remote URL. Also had a
connection reset on the first push which went away on retry. `pnpm install` ran fine
(Node v24.12.0, pnpm 10.28.1, done in about 4.5 seconds).

### Steps to Reproduce

1. Clone the repo and run `pnpm install`
2. Open any of the 14 files listed above and go to the line number
3. You'll see `await` inside a `for...of` loop — the violation is right there in the code
4. Run `pnpm lint` from the root to see the React Doctor rule flag each one

### Reproduction Evidence

The problem is visible statically, no need to run the app. Here's what it looks like
in `add-administrator-wizard.tsx` around line 274:

```typescript
// this is the problem — each call waits for the previous one
for (const roleId of roleIds) {
    setIsSubmitting(true);
    await updateUsersForRoleFunction(roleId, roleData)
        .catch(...)
        .finally(...);
}
```

**Branch:** [tommylava/identity-apps — fix-issue-27874](https://github.com/tommylava/identity-apps/tree/fix-issue-27874)

## Solution Approach

### Analysis

The calls inside each loop are independent — none of them need the result of the
previous one. So there's no reason to run them one at a time. The fix is to collect
them all into a `Promise.all` so they run at the same time.

### Proposed Solution

Replace each `for...of` loop that has `await` inside it with
`await Promise.all(items.map(async (item: Type): Promise<void> => { ... }))`.
Keep all the existing `.catch()` and `.finally()` handlers, just move them inside
the map callback. This repo also has strict TypeScript conventions so I need to
make sure the type annotations on the map callback are explicit.

### Implementation Plan (UMPIRE)

**Understand:**
14 files are running async API calls one at a time inside loops when they could all
run at the same time. The React Doctor rule `async-await-in-loop` caught all of them.

**Match:**
The issue description itself shows the fix pattern:
```typescript
// before
for (const item of items) {
    await someCall(item);
}

// after
await Promise.all(items.map(async (item: ItemType): Promise<void> => {
    await someCall(item);
}));
```
The `CLAUDE.md` in the repo also requires explicit TypeScript types on all variables
and callbacks, so I need to match that style.

**Plan:**
1. Go through each of the 14 files at the listed line numbers
2. Replace the `for...of` loop with `Promise.all(items.map(...))`
3. Keep all the `.catch()` and `.finally()` handlers inside the map
4. Make sure the type on the map callback matches what the loop variable was
5. Run `pnpm lint` to confirm no more React Doctor violations
6. Run `pnpm typecheck` to make sure no new TypeScript errors

**Implement:** Phase III — branch link:
[fix-issue-27874](https://github.com/tommylava/identity-apps/tree/fix-issue-27874)

**Review:**
- Follow the TypeScript conventions in `CLAUDE.md` (explicit types, no `any`,
  `Promise<void>` return type on async callbacks)
- Read `pull_request_template.md` before submitting the PR
- Use a conventional commit message

**Evaluate:**
- `pnpm lint` shows zero `async-await-in-loop` violations after the fix
- `pnpm typecheck` passes with no new errors
- Each changed file still has its `.catch()` and `.finally()` handlers intact

## Testing Strategy

**Linting:**
Run `pnpm lint` — should show zero React Doctor violations for `async-await-in-loop`
after the fix.

**Type checking:**
Run `pnpm typecheck` — should pass with no new TypeScript errors introduced.

**Manual review:**
Go through each changed file and confirm the error handling (`.catch()` blocks) and
cleanup (`.finally()` blocks) are still there and working correctly.

## Pull Request

**PR Link:** Not submitted yet
**Status:** Phase II Complete — implementation starts in Phase III

## Learnings & Reflections

**Technical Skills Gained:**
Learned how to navigate a large TypeScript monorepo, how React Doctor static analysis
works, and the difference between sequential and concurrent async patterns in JavaScript.

**Challenges Overcome:**
Had a 403 on the first push because I cloned the original instead of my fork. Fixed
by updating the remote URL. Also had to figure out how pnpm workspaces are structured
in a monorepo this size.

**What I'd Do Differently Next Time:**
Fork before cloning so the remote is set up correctly from the start.

## Resources Used

- GitHub issue #27874 and React Doctor rule documentation
- `CLAUDE.md` and `CONTRIBUTING.md` in the identity-apps repo
- MDN docs on `Promise.all`
