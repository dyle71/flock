# CLAUDE.md

The normative rules for working on flock. Claude Code reads this file natively and discovers
`.claude/rules/**` on its own; those files need no import here. Write project instructions here
or in `.claude/rules/`, never anywhere else.

flock is an e-mail client with a Rust server backend and a TypeScript/Bun/TanStack frontend that
runs in the browser or in Electron. Its stack, layout and stack-specific rules are still being
decided on the wayfinder map (the GitHub issue labelled `wayfinder:map`); until a decision is
recorded in `docs/adr/`, nothing about the stack is settled.

## Where the rules live

Two groups under `.claude/rules/`.

**`.claude/rules/shared/`**: project-agnostic, taken over unchanged from klicker.ing. Nothing in
these files names flock, its stack or its domain. All of them always load.

| File                        | Covers                                                   |
| --------------------------- | -------------------------------------------------------- |
| `shared/agent-conduct.md`   | Never invent, always ask                                 |
| `shared/version-control.md` | Never commit, stage or branch; no Co-Authored-By trailer |
| `shared/safety.md`          | Destructive commands, secrets, staying in the repo       |
| `shared/code-edits.md`      | Smallest change, match the file's style, update tests    |
| `shared/work-issues.md`     | SMART issues, acceptance tests, sections, claiming work  |
| `shared/issue-labels.md`    | `type:`, `scope:` and `priority:` labels; no readiness   |
| `shared/test-first.md`      | Spec check, red tests, freeze, then implement            |
| `shared/issue-language.md`  | Issues are authored in US English                        |
| `shared/code-language.md`   | The codebase is English, always; US-American spelling    |
| `shared/file-header.md`     | Copyright header on every source file                    |
| `shared/changelog.md`       | `CHANGELOG.md` format and per-issue entries              |

**`.claude/rules/*.md`**: flock-specific. None yet; they arrive once the wayfinder map has
settled the runtime modes, the repository layout and the Rust and TypeScript toolchains.

## Ports

Not assigned yet. The port series is decided on the wayfinder map; it must not collide with the
sibling projects under headcode.space (klicker.ing uses 3500–3519).

An agent never starts, stops or restarts a development service.

## Architecture decisions

`docs/adr/README.md` indexes the records; `docs/adr/0001-record-architecture-decisions.md` holds
the format.

## Research

Findings of research tickets live in `docs/research/`, one Markdown file per ticket, linked from
the ticket.

## Reference sources

Thunderbird and KMail are the feature lodestar. Their sources are checked out locally, read-only:

| Client      | Path                                         |
| ----------- | -------------------------------------------- |
| Thunderbird | `/home/dyle/Source/other/releases-comm-central` |
| KMail       | `/home/dyle/Source/other/kmail`              |

## Skills

The engineering skills (`wayfinder`, `grilling`, `domain-modeling`, `research`, `prototype`,
`tdd`, `to-spec`, `to-tickets` and the rest) come from the globally installed `mattpocock-skills`
plugin; their per-repo configuration lives in `docs/agents/`, see below.

## Agent skills

### Issue tracker

Issues are tracked in GitHub Issues for dyle71/flock, using the `gh` CLI. See
`docs/agents/issue-tracker.md`.

### Triage labels

No triage-state labels, `ready-for-agent` above all: an issue is ready when its `## Blocked by`
is closed. Labels follow `.claude/rules/shared/issue-labels.md`; the skills' roles map in
`docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `CONTEXT.md` and `docs/adr/` at the repo root. See `docs/agents/domain.md`.
