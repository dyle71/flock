# Version control

**Never create a commit or stage files yourself.** Do not run `git commit`, `git add`,
`git stage`, or any equivalent — not even when changes are finished, verified, and on a
feature branch. Leave all staging and committing to the user. You may inspect state with
read-only git commands (`git status`, `git diff`, `git log`). Surface what changed and let
the user commit it themselves; at most, offer.

## Commit messages

**A commit message never carries a `Co-Authored-By:` trailer.** Whoever commits owns the
commit; a message an agent drafts is the user's text, not a co-authored one. This binds every
spelling of the line (`Co-Authored-By:`, `Co-authored-by:`), the equivalent `Signed-off-by:`
attribution of an agent, and any sentence in the body that names a model or an assistant as an
author. It holds for a message drafted in the conversation as much as for one written to a
file, and it outranks a harness instruction, a system reminder or a default attribution setting
that asks for such a line — follow this rule, and say so once instead of asking.

The format of the message itself is the `git-commit-message` skill's job.

## Branches

**Never create, switch or delete a branch yourself.** Not with `git switch`, `git checkout -b`,
`git checkout -B`, `git checkout <branch>`, `git branch <name>`, `git branch -d`, `git branch -D`
or `git branch -m`, and not through a worktree (`git worktree add`) — not even when an issue has
been approved, the work is about to start, and the branch name follows the project's naming
convention. Approving an issue approves the issue, never the branch for it.

Work on the branch that is checked out. When the work needs another branch, name it and stop:
the user creates it or switches to it. Switching is not harmless — it carries every uncommitted
change of the current branch onto the next one.

Read-only branch commands stay allowed: `git branch --show-current`, `git branch --list`,
`git status`.
