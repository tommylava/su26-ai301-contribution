# Contribution 1: Fix async-await-in-loop Performance Issue

**Contribution Number:** 1
**Student:** Thomas Lavadinho
**Issue:** wso2/product-is#27874
**PR:** wso2/identity-apps#10480
**Status:** Phase IV Complete — PR submitted, review comments addressed, awaiting maintainer approval

## Why I Chose This Issue

I switched to this issue from a Session Desktop one because this one is way more approachable. The React Doctor tool already flagged exactly which files and line numbers have the problem, so I didn't have to go hunting through a huge codebase trying to figure out what was wrong. The fix pattern is also clear — it's the same change in multiple places. And it's a real performance improvement, not just a style thing, which made it feel worth doing.

## Understanding the Issue

**Problem Description:** In several files across the identity-apps repo, there are await calls sitting inside for...of loops. This means every async API call has to fully finish before the next one even starts. If all the calls are independent (which they are here), that's just wasted time.

**Expected Behavior:** Independent async calls should all fire at the same time using Promise.all, so the total wait time is just however long the slowest single call takes instead of adding them all up.

**Current Behavior:** Every call waits for the one before it. If each call takes 200ms and there are 5 of them, you're waiting 1000ms instead of ~200ms.

**Affected Components:** 14 files flagged by React Doctor in the features/ directory of wso2/identity-apps:

- features/admin.administrators.v1/wizard/add-administrator-wizard.tsx:279
- features/admin.connections.v1/components/edit/settings/authenticator-settings.tsx:442
- features/admin.connections.v1/components/edit/settings/outbound-provisioning-settings.tsx:239
- features/admin.console-settings.v1/hooks/use-enterprise-login-config.ts:230
- features/admin.console-settings.v1/hooks/use-enterprise-login-config.ts:547
- features/admin.copilot.v1/api/copilot-api.ts:233
- features/admin.copilot.v1/store/actions/copilot.ts:526
- features/admin.flow-builder-core.v1/api/use-resolve-custom-text-preference.ts:140
- features/admin.flow-builder-core.v1/plugins/plugin-registry.ts:180
- features/admin.server-configurations.v1/pages/connector-listing-page.tsx:284
- features/admin.template-core.v1/hooks/use-initialize-handlers.ts:91
- features/admin.template-core.v1/hooks/use-initialize-handlers.ts:114
- features/admin.template-core.v1/hooks/use-submission-handlers.ts:118
- features/admin.template-core.v1/hooks/use-validation-handlers.ts:114

## Reproduction Process

### Environment Setup

I forked wso2/identity-apps to tommylava/identity-apps and cloned it locally. First push attempt failed with a 403 because I had originally cloned the original repo directly instead of my fork — fixed that by updating the remote URL. Also had a connection reset on the first push which went away on retry. Had a Windows long path issue that caused the initial clone to be missing most files — fixed by running git config --global core.longpaths true and re-cloning. pnpm install ran fine (Node v24.12.0, pnpm 10.28.1).

### Steps to Reproduce

1. Clone the repo and run pnpm install
2. Open any of the 14 files listed above and go to the line number
3. You'll see await inside a for...of loop — the violation is right there in the code
4. Run pnpm lint from the root to see the React Doctor rule flag each one

### Reproduction Evidence

The problem is visible statically, no need to run the app. Here's what it looks like in add-administrator-wizard.tsx around line 274:

```typescript
// this is the problem — each call waits for the previous one
for (const roleId of roleIds) {
    setIsSubmitting(true);
    await updateUsersForRoleFunction(roleId, roleData)
        .catch(...)
        .finally(...);
}
```

Branch: tommylava/identity-apps — fix-issue-27874

## Solution Approach

### Analysis

After going through all 14 flagged files I found two categories — real violations where the fix applies, and false positives where the loop is intentionally sequential and changing it would break things.

**Real violations fixed (5 files):**

- add-administrator-wizard.tsx — independent role update calls
- authenticator-settings.tsx — independent authenticator fetch calls
- outbound-provisioning-settings.tsx — independent connector fetch calls
- use-enterprise-login-config.ts (line 547) — independent role patch calls
- connector-listing-page.tsx — independent category load calls

**False positives skipped (9 files):**

- copilot-api.ts and copilot.ts — while loops, not for...of, and each iteration depends on the previous one
- use-resolve-custom-text-preference.ts — already using Promise.all
- plugin-registry.ts — early-exit pattern, needs to stop on the first false
- use-enterprise-login-config.ts (line 230) — sync loop, no await in the loop body
- use-initialize-handlers.tsx, use-submission-handlers.tsx, use-validation-handlers.tsx — ordered processing where each step may depend on the previous one

### Proposed Solution

Replace each real for...of loop with await Promise.all(items.map(...)). Keep all the existing .catch() and .finally() handlers inside the map callback. Follow the repo's TypeScript conventions from CLAUDE.md — explicit types on the map callback and Promise<void> return type.

### Implementation Plan (UMPIRE)

**Understand:** Several files run async API calls one at a time inside loops when they could all run at the same time. The React Doctor rule async-await-in-loop flagged all of them.

**Match:** The issue description itself shows the fix pattern:

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

The CLAUDE.md in the repo also requires explicit TypeScript types on all variables and callbacks, so I matched that style.

**Plan:**
1. Review each flagged file to decide if it's a real violation or a false positive
2. For real violations, replace the for...of loop with Promise.all(items.map(...))
3. Keep all .catch() and .finally() handlers inside the map
4. Make sure type annotations match the repo's TypeScript conventions
5. Run pnpm lint to confirm zero violations remain

**Implement:** ✅ Complete — fix-issue-27874

**Review:**
- Followed TypeScript conventions in CLAUDE.md (explicit types, no any, Promise<void> return type on async callbacks)
- Read pull_request_template.md before submitting the PR
- Used conventional commit message format

**Evaluate:**
- pnpm lint shows zero async-await-in-loop violations ✅
- All .catch() and .finally() handlers preserved in changed files ✅

## Implementation Notes

**Files I actually changed:**

- add-administrator-wizard.tsx — replaced for...of roleIds in assignUserRoles, moved setIsSubmitting outside the loop since it only needs to be called once
- authenticator-settings.tsx — replaced for...of authenticators in fetchAuthenticators with Promise.all, added try/catch returning null on failure with .filter(Boolean) after Promise.all to handle fetch failures gracefully
- outbound-provisioning-settings.tsx — replaced for...of connectors in fetchConnectors with Promise.all, added try/catch returning null on failure with .filter(Boolean) after Promise.all to handle fetch failures gracefully
- use-enterprise-login-config.ts — replaced for...of consoleRoles in removeConfiguration with Promise.all, removed the unused patchPromises array
- connector-listing-page.tsx — replaced for...of categories in resolveDynamicCategories with Promise.all and added .filter(Boolean) to remove any null results

**Biggest challenge:** Figuring out which files were real violations vs false positives. The linter fires on any await inside any loop — including while loops and loops where sequential execution is actually what you want. I had to read each file carefully before touching anything.

**Review feedback addressed:** The CodeRabbit bot flagged that Promise.all could hang indefinitely if a fetch failed and the error was caught but the promise never resolved. Fixed by adding try/catch inside the map callbacks for authenticator-settings.tsx and outbound-provisioning-settings.tsx, returning null on failure and filtering nulls out after Promise.all resolves.

## Testing Strategy

**Linting:** Run pnpm lint — zero async-await-in-loop violations after the fix ✅

**Type checking:** Run pnpm typecheck — confirms no new TypeScript errors were introduced.

**Manual review:** Went through each changed file to confirm the .catch() and .finally() handlers are still there and the logic is equivalent to the original.

## Pull Request

**PR Link:** wso2/identity-apps#10480
**Status:** Phase IV Complete — PR submitted, review comments addressed, changeset added, awaiting maintainer approval

## Learnings & Reflections

**Technical Skills Gained:** Learned how to navigate a large TypeScript monorepo, how React Doctor static analysis works, and how to tell the difference between sequential and concurrent async patterns. Also learned that linter rules have false positives and you have to actually read the code before applying a fix. Also got experience responding to code review feedback from both a bot and a maintainer.

**Challenges Overcome:** Windows long path issue caused the initial clone to be missing most files — fixed with git config --global core.longpaths true. Also had to sort through 14 flagged files and figure out which 5 actually needed changing. After submitting the PR, had to address CodeRabbit feedback about Promise.all hanging on failed fetches and add a missing changeset file.

**What I'd Do Differently Next Time:** Enable long paths in git before cloning on Windows. Also fork before cloning so the remote is set up correctly from the start. Check if the project requires a changeset file before submitting the PR so it's included from the start.

**Resources Used:**
- GitHub issue #27874 and React Doctor rule documentation
- CLAUDE.md and CONTRIBUTING.md in the identity-apps repo
- MDN docs on Promise.all
