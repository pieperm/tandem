# Project overlay — <project name>

Copy this file to `CUSTOMIZE.md` (which is gitignored) and fill it in. `SKILL.md` stays
generic and reads the overlay for everything below, so the skill can be updated from
upstream without losing your project's specifics.

Every section is optional. A heading you leave empty means "no project convention here" —
the skill falls back to its generic behavior. Delete the guidance text as you fill each in.

Mark anything time-sensitive *(as of YYYY-MM-DD)* so a later reader knows to re-verify.

> **If reading your tracker needs an MCP tool,** note that `SKILL.md` deliberately declares
> no `allowed-tools`, so whatever tools the session has are available. Don't add an
> `allowed-tools` list unless you intend to maintain it.

---

## Project

One paragraph: language, build system, frameworks, how it deploys. Enough that a reviewer
knows what kind of code they're about to read.

## Ticket tracker

**System:** Jira / GitHub issues / Linear / none. Name the tool or MCP the skill should
call, and any routing rule (e.g. "route through <meta-tool>, never call `jira_*` directly").

**Key pattern:** e.g. `ABC-*`, and where it's found — branch name, commit subject prefix,
PR title.

**Reading the issue:** the exact call or command, with whatever field projection keeps the
payload small.

**Comments:** often a separate call from the issue itself. Note it if so — this is where
real scope frequently lives.

**Acceptance criteria:** where they actually live. Often *not* the description — a checklist
app, a custom field, a task list in the PR body, a linked doc. Record the mechanism once so
nobody has to rediscover it, along with anything it does or doesn't expose (per-item checked
state, aggregate counts).

**Gotchas:** field names that silently return nothing, flags that belong to a different
endpoint, decoy keys, sibling calls worth knowing.

## Review output

Where review documents go, if not the default `reviews/<TICKET>/<short-sha>.md`.

## Git

**Ref namespace for fetched PR heads:** defaults to `refs/pr-review/pr<n>`. Override if it
collides with something.

**Noise commits.** Subjects the skill should group into one line and not review — CI
version bumps, generated lockfiles, formatter passes. Give the literal subject or a pattern.

**Commit order.** Anything known about how commits land here: whether the branch is
routinely rebased, whether the first commit is typically the implementation or a review
response.

**Etiquette.** Whether to commit the review document, push, or leave git alone.

## Code layout

| Kind | Pathspec |
|---|---|
| Production source | |
| Unit tests | |
| Integration tests | |
| Config | |
| Deployment | |
| Docs | |

Note anything non-obvious: per-environment config files, generated directories to skip,
modules with different conventions.

## Sweep patterns

Ready-to-run greps for step 8, with the changed symbol marked for substitution. The skill
knows *what* to look for; this section says how to look for it in this codebase.

**Callers of a changed signature or contract:**

```bash
git grep -n "<changedSymbol>" <ref> -- '<source pathspec>' | grep -v "<the changed file>"
```

**Sibling instances of a fixed pattern.** List the shapes that recur here — the bug classes
this codebase produces more than once.

**Silent contract changes.** The ones the compiler or type checker won't catch: error vs.
empty, nullability, thrown type, iteration order. Name the downstream constructs that would
silently change behavior.

**Config that encodes the same fact:**

```bash
git grep -rn "<propertyName>" <ref> -- '<config pathspec>' '<deploy pathspec>'
```

**Tests that exist but weren't updated:**

```bash
git grep -ln "<ChangedSymbol>" <ref> -- '<test pathspec>'
```

## CI

How to check the build for a specific SHA, if not `gh pr checks <n>`.

## Review lessons

Accumulated mistakes worth not repeating — each one a wrong or missed review statement that
actually shipped. The most valuable entries are *Blast Radius* claims that turned out to
be false, since a false entry costs the author more than a missed one.

- **<ticket> — <one-line label>.** What was claimed, what was actually true, and the check
  that would have caught it.