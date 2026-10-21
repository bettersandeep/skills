---
name: adversarial-review
description: Adversarial review of a GitLab MR you review for someone else. Three agents attack the change with your principles, then try to refute each other. You get only the findings that survive, and an offer to post them as draft MR comments.
disable-model-invocation: true
argument-hint: <GitLab MR URL>
---

# Adversarial review

The user is the reviewer of someone else's MR.
Three reviewer agents attack the change.
Then each agent tries to refute the findings of the other two.
You judge what survives, report it to the user, and offer to post draft comments.

All agents use [principles.md](principles.md).

## Finding rules

A finding has all of these:

- `file:line` in the MR head.
- Evidence: the quoted code lines, plus any other code the claim depends on (a caller, an existing helper, a config value).
- A **repro** or a **replacement**.
  A repro is a concrete input or state and the wrong result it causes.
  A replacement is the code that should stand instead (or the deletion), and why it handles the same cases.
- The principle it breaks, by number.
- Severity: `blocker` (wrong result, data loss, security, prod risk), `should-fix` (principle break with a real cost), or `nit` (taste).

A claim with no evidence and no repro or replacement is not a finding.
Every "might" or "could" needs a repro behind it.
If `base_sha` differs from `start_sha`, the branch is behind the target branch.
Before you claim that something is missing, check the target branch for it.

## Steps

### 1. Load the MR

Parse the URL into host, project path, and MR iid.
Work in your scratchpad directory.

```sh
export GITLAB_HOST=<host from the MR URL>
P=$(jq -rn --arg p "<group/project>" '$p|@uri')
glab api "projects/$P/merge_requests/<iid>" > mr.json
glab api "projects/$P/merge_requests/<iid>/discussions?per_page=100" > discussions.json
```

`mr.json` has `diff_refs` (`base_sha`, `start_sha`, `head_sha`), `target_branch`, title, and description.

Get the code at the MR head:

- If the current directory is a clone of the project: `git fetch origin <target_branch> merge-requests/<iid>/head`, then `git worktree add <scratch>/mr-<iid> <head_sha>`.
- Else: `glab repo clone <group/project> <scratch>/mr-<iid> -- --filter=blob:none`, fetch the same refs, and `git checkout <head_sha>`.

Write the diff: `git -C <scratch>/mr-<iid> diff <base_sha> <head_sha> > <scratch>/mr-<iid>.diff`.

Write the **intent** in 3 to 5 lines: the problem, and what the MR says it does.
Use the title, the description, and the linked ticket if the description names one.
The necessity lens judges every unit against this intent.

Done when: the worktree is at `head_sha`, the diff file is not empty, and the intent is written.

### 2. Attack

Spawn three `general-purpose` agents in one message so they run in parallel.
Each agent gets one lens.
Each agent applies all principles, but spends most effort on its lens.

| Lens | Prefix | Principles | Job |
|---|---|---|---|
| necessity | N | 1, 2, 9 | For each added unit: what breaks if we delete it? Does it already exist in the repo, stdlib, or a dependency? Grep the worktree to prove it. |
| correctness | C | 5, 6, 7, 8, 10 | Break it. Trace callers and data flow outside the diff. Run the code or the tests when it is cheap. |
| simplicity | S | 3, 4 | Write the replacement. It must be correct on every case the original handles. |

Prompt for each agent:

```
You are the <lens> reviewer in an adversarial review of MR <url>.
Intent: <intent>
Code at MR head: <worktree>. Read and run code there. Do not commit or push.
Diff: <diff file>
Principles: <absolute path to principles.md>. Read it first.
Your lens: <job from the table>. Principles to focus on: <numbers>.
Read past the diff: callers, tests, existing helpers. Every claim cites code you read or ran.
<paste the Finding rules section>
Return a numbered list of findings. Each has: id (<prefix>-1, <prefix>-2, ...), file:line, claim, evidence, repro or replacement, principle, severity.
Done when: you checked every added or changed unit in the diff against your lens.
After the findings, list each unit you checked and found clean, one line each.
```

Done when: all three agents returned.

### 3. Cross-examine

Load SendMessage (`ToolSearch` with `select:SendMessage`).
Continue each agent and send it the findings of the other two agents:

```
Here are the findings of the other two reviewers. Try to refute each one.
Re-read the cited lines. Run the repro, or trace it through the real code.
Check if the code handles the case somewhere else.
For a replacement, check that it is correct on every case the original handles, and that it is simpler in fact.
Give each finding a verdict with evidence: upheld, weakened (give the corrected claim or severity), or refuted.
Also say when the fix costs more than the problem.
Done when: every finding has a verdict with evidence.
```

If SendMessage fails, spawn a fresh agent with the same lens, the round 1 prompt header, and this text.

### 4. Judge

You decide each finding.
You have the claim from its author and two verdicts from the other agents.

- Merge duplicate findings across agents.
- Read the cited lines yourself, and check the repro or replacement.
- A finding that a verdict refuted with evidence that holds against the code: `drop`.
- A confirmed `blocker` or `should-fix`: `fix`.
- A confirmed finding that is low cost or a matter of taste: `consider`.
- A finding already raised in an open discussion in `discussions.json`: mark it "already raised".

Done when: every finding has a verdict, and you checked every `fix` and `consider` against the code yourself.

### 5. Report

Write the report in ASD-STE100.
Use plain hyphens, never em dashes.

- One line: MR, intent, and your call (approve, approve with changes, or request changes).
- `fix` items, then `consider` items, by severity.
  Each item: id, `file:line`, claim, repro or replacement, and the verdict trail (for example "C: upheld, S: weakened to nit").
- Dropped findings, one line each, with the reason.
  This tells the user which comments are not worth the time of the MR author.

### 6. Offer to post

Ask the user which item ids to post.
Post nothing without an answer.

Write each comment for the MR author: the problem, the repro or replacement, and the request.
Keep it short, in ASD-STE100, with no em dashes.
The comments go out in the user's name: read `~/VOICE.md` if it exists.

Post each one as a draft note, so the user publishes all of them as one review:

```sh
# body.json
{"note": "<text>", "position": {"position_type": "text",
  "base_sha": "<base_sha>", "start_sha": "<start_sha>", "head_sha": "<head_sha>",
  "old_path": "<path>", "new_path": "<path>", "new_line": <N>}}

glab api "projects/$P/merge_requests/<iid>/draft_notes" -X POST \
  -H "Content-Type: application/json" --input body.json
```

- `new_line` is the line number in the head file, and the line must be inside a hunk of the diff.
  For a removed line, use `old_line` instead.
  For a finding with no line in the diff, send the note with no position.
- Send JSON with the `Content-Type` header.
  `glab api -f "position[...]"` form fields are dropped with no error.
- Check that `position` in the response is not null.
  If it is null, delete that draft (`DELETE .../draft_notes/<id>`) and post it again.

Tell the user that the drafts are pending, and that "Submit review" in the MR publishes them.

### 7. Clean up

Remove the worktree (`git worktree remove`) or the clone.
