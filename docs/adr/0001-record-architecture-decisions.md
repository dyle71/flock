# 0001. Record architecture decisions

- Status: Accepted
- Date: 2026-11-11

## Context

flock starts from a wayfinder map (the GitHub issue labelled `wayfinder:map`) whose tickets
settle, one by one, why the application will be built the way it is. Those reasons would
otherwise live only in closed issues and in the dev's head. A coding agent that cannot see that a
decision was made deliberately may propose to reverse it.

The sibling projects under headcode.space record such decisions in `docs/adr/` in one shared
format; klicker.ing is the model project. flock keeps that format.

## Decision

Architecture decisions are recorded in `docs/adr/`, one file per decision, in Michael Nygard's
format: a title, a status, and the sections Context, Decision and Consequences.

- Files are named `NNNN-kebab-title.md`, numbered in order starting at `0001`, and never
  renumbered. `docs/adr/README.md` lists every record with its status.
- Records are append-only. A reversed decision gets a new record whose status line reads
  `Accepted, supersedes [NNNN](NNNN-kebab-title.md)`; in the old record, only the status line
  changes, to `Superseded by [NNNN](NNNN-kebab-title.md)`.
- A record of a decision that has not been carried out yet carries the status `Proposed`. It
  becomes `Accepted` in the branch that starts carrying it out, and nothing else about the
  record changes. A record that is never carried out becomes `Rejected`, with one sentence
  under the status line saying why.
- A decision settled on the wayfinder map gets a record when all three hold: it is hard to
  reverse, it would surprise a reader without context, and it came out of a real trade-off.
  The ticket that settled it links the record.
- Agents read the records on demand. `CLAUDE.md` links `docs/adr/` and does not import it.

## Consequences

- The reasons for a decision live next to the code and are reviewed in the same pull request.
- The records do not grow the context of every agent session.
- Records describe decisions; they do not instruct. `CLAUDE.md` and `.claude/rules/` remain the
  single source of instructions.
