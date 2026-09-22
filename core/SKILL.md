---
name: pr-review-doc
description: Produce a durable file-by-file review document for a pull request, written to reviews/<TICKET>/<commit>.md. Runs in report mode (one pass, whole document) or interactive mode (walk the files one at a time, answering the reviewer's questions and filling the document in as each file is confirmed). Use when the user asks to review a PR file by file, wants to be walked through a PR, wants the review written down or saved, references a ticket alongside a PR, or asks to re-review a PR after new commits. Covers pulling the ticket for scope, resolving the PR head, deriving per-file commit history, judging what the PR should have touched but didn't, and the format the review must follow. For a findings-only pass with no document, use /code-review instead.
---

# PR Review Document

Produces one markdown file per (ticket, reviewed commit), so a ticket reviewed at several points keeps every review side by side.

**Output path:** `reviews/<TICKET>/<short-sha>.md`, unless the overlay says otherwise. Ticket comes from the branch name or commit subjects; the overlay gives the key pattern. Short SHA is the PR head at review time, 8 chars.

This is not `/code-review`. That skill hunts for defects and reports findings. This one explains **what changed and why**, file by file, with defects as notes alongside. A reader who never saw the PR should be able to follow the change from this document alone.

**Two modes, same document.** *Report* analyses everything and writes it in one pass. *Interactive* walks the files one at a time so the reviewer can ask how something works or why it was built that way, filling the document in as each file is confirmed. Ask which one at step 3.

## Reading this file

- **Invariants** — properties of git or `gh`. Rely on these.
- **Project specifics** — anything that depends on the language, repo layout, tracker or CI lives in the overlay, not here. This file says *what* to look for; the overlay says how to look for it in this codebase.
- **Dated context** — marked *(as of YYYY-MM-DD)*. Verify before relying on it.

If something here no longer matches what you observe, say so and offer to update this file — or the overlay, if the divergence is project-specific.

## Workflow

### 0. Read the project overlay

Read `CUSTOMIZE.md` next to this file. It holds the ticket-key pattern, how to reach the tracker, the ref namespace, which commits are noise, the source/test/config pathspecs, the sweep greps for step 8, and the review lessons this repo has already paid for.

If it's absent, say so once, work from the generic instructions below, and offer to create one from `CUSTOMIZE.template.md` at the end — the things you had to infer are exactly what belongs in it.

### 1. Resolve the target

```bash
gh pr view <n> --json headRefName,headRefOid,baseRefName,mergeable,mergeStateStatus
```

Fetch the head into a **named ref**, never bare `FETCH_HEAD`:

```bash
git fetch origin <headRefName>:refs/pr-review/pr<n> --force
```

**`FETCH_HEAD` is clobbered by any later fetch.** A `git fetch origin main` midway through a review silently repoints `FETCH_HEAD` at main, and the next diff comes back empty — which reads as "the branch is already merged" rather than as a mistake. This has happened. Use a named ref and the problem disappears.

`<ref>` below means this ref. The overlay may name a different namespace.

### 2. Pull the ticket

The ticket is what makes the *Blast Radius* section possible — without the intended scope you can only describe the diff, not judge it.

Resolve the key from the branch name or the commit subjects, using the pattern in the overlay. How you then read it depends on the project:

- **GitHub issues:** `gh issue view <n> --json title,body,state,comments,labels`
- **Jira via MCP:** typically `jira_get_issue`, then `jira_list_comments` as a **separate call**. Always project the fields you need; unprojected issue payloads are enormous.
- **No tracker reachable, or the key can't be resolved:** continue, and say in the document that scope was judged from the code alone.

**Acceptance criteria are often not in the description.** They may live in a checklist app, a custom field, a task list in the PR body, or a linked doc. The overlay should say where; if it doesn't, find out once and offer to record it. **These are the section's most valuable input**: an unchecked criterion that the PR doesn't address is exactly a *Blast Radius* entry, sourced from the ticket rather than from your judgement.

**A missing description is itself a finding.** If the ticket states no scope at all, the summary is the only statement of intent and *Blast Radius* rests entirely on your reading of the code. Say that explicitly rather than quietly reviewing as though the ticket had justified the scope, and note that defining "done" is outstanding.

What to take from the ticket: the **stated problem**, the **acceptance criteria**, and whether **sibling tickets** already own work this PR omits. A deliberate omission tracked elsewhere is a note, not a defect — and only the tracker can tell you which it is. Carry the criteria into the review as the yardstick: for each unmet one, decide whether this PR delivers it, and if not, whether that's deferred or missed.

### 3. Ask which mode

**Always ask. Never assume.** Use `AskUserQuestion` with exactly these two options:

- **Report** — analyse everything, write the whole document, present the findings. One pass, no interruptions.
- **Interactive** — walk the files one at a time. The reviewer can ask how something works or why it was built that way, and the document is filled in file by file as each is confirmed.

Ask here rather than at the very first turn: steps 0–2 are cheap reads and nothing has been written yet, so by now you can state **how many files are in the diff** and **the ticket summary**, which is what the reviewer needs to choose. A 3-file PR rarely wants interactive; a 20-file one usually does.

If the user already said which they want ("walk me through it", "just write it up"), skip the prompt and honour it.

Steps 4–7 run identically in both modes. Mode changes only how steps 8–10 and the writing are paced — see **Interactive mode** below.

### 4. Read the existing review conversation

PR-level comments and inline review comments are **different APIs**. `gh pr view --json comments` returns only the former and is routinely empty on a PR that has plenty of inline discussion:

```bash
gh pr view <n> --json reviews --jq '.reviews[] | "\(.author.login) [\(.state)] \(.submittedAt)\n\(.body)"'
gh api repos/:owner/:repo/pulls/<n>/comments --jq '.[] | "\(.user.login) @ \(.created_at) on \(.path):\(.line // .original_line)\n\(.body)"'
```

An open reviewer thread outranks everything else in the document. Answer it explicitly in the Notes of the file it was left on, and say plainly whether the PR resolved it, partly resolved it, or only appears to.

### 5. Establish commit order

```bash
TZ=UTC git log --reverse --format='%h  %ad  %an  %s' --date=format:'%Y-%m-%dT%H:%M:%SZ' <merge-base>..<ref>
```

- **Committer dates are worthless after a rebase** — every commit on a rebased branch shares one timestamp. `gh pr view --json commits` reports exactly that date.
- **Author dates can collide too** when a working session is split into several commits. Then only parent-to-child order (`git log --reverse`) means anything.
- Author dates *do* reliably separate the original implementation from later review responses. Compare them against the reviewer's `submittedAt`. Whether the first commit listed is implementation or a review response is a repo habit — the overlay may say.
- Group the noise commits named in the overlay (CI version bumps, generated files, formatter passes) into one line at the end and review nothing.

### 6. Per-file inputs

File list, and hunk headers carrying **new-file** line numbers:

```bash
gh pr diff <n> --name-only
gh pr diff <n> | awk '/^\+\+\+ /{f=substr($0,5); sub(/^b\//,"",f); sub(/\t.*$/,"",f)} /^@@/{print f"  "$0}'
```

Track the file from the `+++` line, not from `diff --git`. The `diff --git a/x b/x` form
needs field splitting to get the path, so `$3` truncates at the first space — a path like
`my module/Thing.java` becomes `my` — and `$3` is the `a/` side, which names the *old* file
and so mislabels every rename, exactly when the new-file line numbers matter most. The `+++`
line has one path, taken as a substring rather than a field, so spaces survive; `b/` is
stripped, a trailing tab (git appends one when the path contains spaces) is dropped, and
deleted files fall out as `/dev/null`.

Per-file history, oldest first:

```bash
git log --reverse --format='  %h %s' <merge-base>..<ref> -- <path>
```

### 7. Line numbers come from the head file, not the diff

Hunk headers give you a starting point; anchor the real ranges by reading the file at the head commit.

```bash
git show <ref>:<path> | grep -n '<declaration or signature>'
```

**The hunk is not the unit of review.** A hunk that spans 80 lines usually crosses several declarations and contains several unrelated changes; one hunk routinely becomes three or four sections. Grep the head file for the declarations the hunk covers — that's what gives you the ranges to split on. See *Grouping the diffs within a file* for where the boundaries fall.

### 8. Map the blast radius

The diff tells you what changed. This step is how you find what *should* have. Work outward from each changed symbol — do this before writing, because it usually sends you back to read more code.

Each sweep below is a question. The overlay holds the greps that answer it in this codebase; if it doesn't, write them from the pathspecs it gives and offer to save them.

**Who calls every changed signature or contract?** A function whose return type, nullability, error type or emptiness semantics changed can break callers that still compile.

The high-value case is a **silent contract change** — one the compiler or type checker says nothing about. A method that starts erroring where it used to return empty, a field that starts arriving null, a collection whose order stops being stable: every downstream construct that depended on the old behavior is now a behavior change with no diff and no compile error. These are the entries this section exists for.

**Does the fixed pattern appear anywhere else?** If the fix was "don't cache this", "move this blocking read off the event loop", "make this eviction atomic", look for the same shape elsewhere. A second instance of the fixed bug is the single most useful thing this section can surface. The overlay's list of recurring bug shapes is the place to start.

**What config, deployment or docs encode the same fact?** A new or changed setting, a tunable whose meaning shifted, a timeout the change now depends on. Check whether every environment that needs the value has it, and whether deployment manifests, per-environment config, or docs still describe the old behavior.

**Which tests exist but weren't updated?** A test asserting the old contract that still passes is a gap, not a pass.

**Which acceptance criteria from step 2 does no file in the diff address?**

Two rules keep this section honest:

- **Verify the file is really untouched and really affected.** Confirm it's absent from `gh pr diff <n> --name-only`, then read enough of it to state the concrete consequence. A file you merely suspect is not an entry.
- **Distinguish missed from deferred.** If a sibling ticket owns it, or the PR description says so, it's context, not a defect. Check before asserting — a tracker search is cheap.

If nothing survives this sweep, say so. "Nothing missing" earned by a real sweep is a useful review statement; an omitted section is ambiguous.

### 9. Verify before asserting

Two failure modes, both of which have produced wrong review text:

- **Reading the current file when describing an early commit.** A function's behavior at commit 3 is often not its behavior at HEAD. `git show <sha>:<path>` when the claim is about a specific commit.
- **Believing a comment.** A doc comment that says "every such environment sets X" is a claim to check, not a fact. Go read the config. Such a claim has been wrong before — the code was safe for an entirely different reason, and the comment would have misled the next person.

Read the overlay's **Review lessons** before writing. Those are the mistakes this repo has already produced, and they are cheaper to read than to repeat.

Attribute changes to commits with `git log -G'<regex>' <range> -- <path>`. Use `-G`, not `-S`: `-S` counts occurrences, so a string that was *edited* rather than added or removed shows nothing.

### 10. Sanity check

Prefer CI over a local build — it's authoritative for the exact SHA and costs nothing:

```bash
gh pr checks <n>
```

Do not check out the PR branch in the user's working copy. Read everything through `git show <ref>:<path>`.

## Format

Files in reading order — root-cause change first, then what depends on it, then tests, then config, then noise commits. Not alphabetical, not diff order. *Blast Radius* comes after the last touched file and before the Summary, so the reader has the whole change in mind before being asked what's absent from it.

### Naming a file section

**The heading is the file name, not the path.** The path is repeated clutter in a document where the file name is the thing being discussed; the full path goes on the line below, where it's available without competing for attention.

**Add path segments only to break a tie.** If two or more files *in this diff* share a name, take segments from the right until each is unique — `build.gradle` appearing twice becomes `api/build.gradle` and `worker/build.gradle`, not the full paths. Only the colliding files get lengthened; everything else stays bare. Judge collisions against the diff's file list, not the whole repo: a `MyFile.java` that appears once in the PR needs no qualification even if the repo holds five.

### Grouping the diffs within a file

**One diff section per declaration.** A hunk that touches three methods is three sections, not one — split at the declaration boundaries even when the changed lines are contiguous. A reader assesses one method at a time, and a single What/Why stretched over three of them has to generalize, which makes the explanation vague exactly where it should be specific. *Declaration* is the named thing the diff sits in: a method, a class, a field, an enum constant, a top-level function.

**A declaration is the ceiling, not the floor.** Two unrelated changes inside one method are still two sections. The rule caps how much a section may cover; it doesn't stop you splitting finer.

**The exception is a change that repeats.** When the same mechanical edit lands in many declarations — a rename applied throughout, a signature updated at every call site, a formatting pass — one section covering the group says more than twenty near-identical ones. Give the range, say how many declarations it covers, and call out anything that varies between them.

**Imports get no section.** An import block is its own hunk in nearly every diff, so reporting it adds a paragraph per file that says only what the code using it already says. Skip it. Whatever the new dependency enables belongs in the section for the change that needed it, not in a section of its own.

Read them regardless — the import list is the fastest way to see what a file now depends on, and it often points at the callers and layers worth checking in step 8. It's evidence, not a finding.

Give imports their own section only when the imports themselves are the story:

- a dependency the module didn't have before, especially a third-party one
- an import that crosses a layer or module boundary the codebase deliberately keeps apart
- the wrong one of two similarly named types — `java.sql.Date` for `java.util.Date`, `javax.*` in a project that has moved to `jakarta.*`
- a wildcard import where the convention is explicit ones, or a static import that hides where a name came from
- an import left behind for something this PR deleted

**Each diff is its own `###` heading, titled with its line range.** Everything under that heading belongs to that diff and nothing else — What, Why, and any notes about it. The heading is the group boundary, and it's what lets a reader tell at a glance where one explanation ends and the next begins.

**File-level Notes and History are `###` headings too**, siblings of the diffs rather than trailing paragraphs. Left unlabelled at the end, they read as belonging to the last diff — the exact confusion this structure exists to prevent. A note about one diff goes under that diff; only what's true of the file as a whole goes in *Notes — whole file*.

**Don't number the diffs.** Line ranges already order themselves, and ordinals would have to be renumbered every time a diff is inserted or split.

````markdown
# <TICKET> — <short change title>

| | |
|---|---|
| **PR** | [#<n>](<url>) — `<head branch>` → `<base>` |
| **Reviewed at** | `<full sha>` |
| **Review date** | <YYYY-MM-DD> |
| **CI** | <status> |
| **Ticket** | [<TICKET>](<url>) — <summary> (<status>) |
| **Acceptance criteria** | <progress, e.g. "1 of 6 met", or "none stated on the ticket" — see *Blast Radius*> |

Line numbers refer to the file contents at `<short sha>`.

---

## `ChangedFile.java`

*`path/to/ChangedFile.java`*

### Lines X–Y

**What:** <the change, in mechanical terms — what the code does now that it didn't before>

**Why:** <the reason it exists; the failure it prevents or the constraint it satisfies>

- <a note about this diff: feedback, defect, risk, or something a future reader needs>

### Lines X–Y

**What:** ...

**Why:** ...

### Notes — whole file

- <only what applies to the file rather than to one diff — omit the section entirely if there is nothing>

### History

- `<sha>` <one line: what this commit did to this file>

---

## Blast Radius

<One line stating what the section covers and what it was judged against — the ticket's criteria, or the code alone if the ticket stated no scope.>

### `UntouchedFile.java`

*`path/to/UntouchedFile.java`*

**Why it's affected:** <the concrete consequence — which changed contract or pattern reaches this file, and what happens there now>

**Verdict:** Missed | Deferred (`<TICKET>`) | Side effect, no change needed

**Notes:**

- <what to do about it, or why nothing is needed>

### <Unmet acceptance criterion, quoted from the ticket>

**Why it's affected:** <what the PR does and doesn't deliver against it>

**Verdict:** Missed | Deferred (`<TICKET>`)

---

## Summary

<Ranked list of what to act on, most significant first, each naming file:line.>

### Commit history for this review

| Commit | Author | Summary |
|---|---|---|
````

## Writing rules

- **What is mechanical, Why is the reason.** If Why restates What in different words, delete it and find the actual motivation — usually the failure being prevented.
- **Notes are optional at both levels.** A clean diff gets no bullets and a clean file gets no *Notes — whole file* section. Padding every diff with a nit trains the reader to skim.
- Notes carry the defects. State the failure concretely — inputs, then wrong outcome. "Could be racy" is not a note; "two callers both holding token X evict each other's replacement, so one 401 becomes N logins" is.
- **Say when something is good.** A review that only lists problems misrepresents the change and is less useful to the author.
- Call out an issue that a *later commit in the same PR* already fixed only when the reader would otherwise raise it. Then name the fixing commit.
- Distinguish "the code is wrong" from "the comment is wrong" from "the test doesn't prove what it claims". These need different fixes and reviewers routinely conflate them.
- Tests get the same What/Why/Notes treatment. The most useful test note is a **gap** — the case the change motivated and nothing covers.
- Keep History one line per commit, phrased as what it did to *that file*.

For *Blast Radius* specifically:

- **Every entry names a real file or a quoted criterion**, never a category. "Error handling elsewhere may need updating" is not an entry; a named file with the consequence spelled out — which changed contract reaches it, and what now happens there — is.
- **Always give the verdict.** Missed, deferred with the ticket key, or side-effect-no-action. An entry without one reads as an accusation and the author can't act on it.
- **No History block** — these files have no commits in this PR. That absence is the point.
- Keep it short. This section earns its place by being selective; five speculative entries bury the one that matters.
- If the sweep found nothing, write one line saying so and what you checked. Don't delete the section — a reader can't tell an empty sweep from a skipped one.

## Interactive mode

Same document, built one file at a time. The reviewer's questions are part of the process, and what they surface belongs in the file.

### Write the skeleton first

Before presenting anything, create `reviews/<TICKET>/<sha>.md` containing only:

- the completed header table and the "Line numbers refer to…" line
- one heading per file, **in reading order**, named per *Naming a file section* above, each with its path line and then `<!-- pending -->`
- empty `## Blast Radius` and `## Summary` headings

Two reasons this comes first: the reviewer can see the planned order and reorder it before you start, and an interrupted session leaves a valid partial document instead of nothing.

### The loop

For each file in reading order:

1. **Present the review in chat** — the same per-diff sections, Notes and History the report would contain. Do **not** write it to the file yet.
2. **Invite questions, and answer them properly.** "How does this work" and "why was it done this way" are the point of this mode. Read whatever it takes — the commit that introduced the line, a caller two files away, the test that pins it. Answer at the depth asked, in chat.
3. **Ask whether to move on.** A plain question is right here; `AskUserQuestion` on every file is heavy.
4. **On confirmation, write that file's section** into the markdown, replacing its `<!-- pending -->` marker — including anything the conversation changed. Then move to the next file.

After the last file, run the step 8 sweep and present *Blast Radius* for confirmation the same way, then the Summary.

### Rules

- **Only confirmation advances.** A question about the current file is not confirmation. Neither is silence, an ambiguous "ok", or a remark you can't classify — ask again rather than guessing. Advancing early loses the reviewer's place.
- **Write on confirmation — not before, and not batched at the end.** If you defer all writes, an interrupted session produces nothing, which defeats the skeleton.
- **Fold the Q&A back in.** If an answer was non-obvious, it becomes a Note. The question itself is evidence the code's intent isn't self-evident, which is sometimes the finding — "this behavior deserves a comment" is a legitimate note, sourced from a real reader being confused by it.
- **Never revise a written section silently.** If a later file changes your view of an earlier one, say so and amend it explicitly.
- **Let them steer.** Skipping a file, jumping ahead, going back to amend a finished section, or switching to report mode for the remainder are all fine. Keep the document consistent with whatever path was taken.
- **The file is the progress record.** Remaining `<!-- pending -->` markers are the work left. On a resumed session, read the document to find where you were rather than asking.

## Writing the file

Create the ticket subdirectory if absent. Do not overwrite an existing `<sha>.md` without saying so — a second review of the same SHA usually means the first should be read, not replaced.

In **report** mode the file is written once, at the end. In **interactive** mode it is created as a skeleton up front and amended after each confirmation; use `Edit` to replace one `<!-- pending -->` marker at a time rather than rewriting the whole file, so an earlier section can't be lost to a bad regeneration.

**If running as a background job**, writes to the shared checkout are rejected until the session isolates. `EnterWorktree` first; the file then lands under `.claude/worktrees/<name>/reviews/...`, which is a real path the user can copy from. Follow the overlay on whether to commit; when it says nothing, don't commit unless asked.

## Keeping this skill current

Tell the user when reality diverges from these files, and offer to update them. **Route the update to the right file:** anything about this repo's tracker, layout, language, CI or accumulated mistakes belongs in `CUSTOMIZE.md`, not here. Only change `SKILL.md` when the process itself is wrong.

Worth flagging to the overlay:

- The tracker's API changing shape, or acceptance criteria moving somewhere else again.
- A new class of noise commit that should be grouped rather than reviewed.
- A new recurring bug shape worth adding to the step 8 sweeps.
- A *Blast Radius* entry that turned out to be wrong. Those are worth recording as a review lesson, since a false entry costs the author more than a missed one.
- A repeated review-writing mistake, for the same reason.

Worth flagging to this file:

- A `gh` subcommand or field changing shape.
- A step that was consistently done out of order, or one that never earned its cost.
- Friction in interactive mode — a confirmation phrasing that got misread as "next file", a file order that was reordered every time, or a question type that kept recurring and should just be answered pre-emptively in the What/Why.
- Anything you had to infer because the overlay had no section for it. That's a gap in `CUSTOMIZE.template.md`.