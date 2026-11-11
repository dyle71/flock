# Triage Labels

The skills speak in terms of five canonical triage roles. This repo does not use triage-state
labels: an issue is ready when everything in its `## Blocked by` section is closed, and its
labels follow `.claude/rules/shared/issue-labels.md`. This file maps the skills' roles onto that.

| Label in mattpocock/skills | Label in our tracker | Meaning                                         |
| -------------------------- | -------------------- | ----------------------------------------------- |
| `needs-triage`             | none                 | Not used                                        |
| `needs-info`               | `type: suggestion`   | Still holds an open question; not implementable |
| `ready-for-agent`          | none                 | Not used; never apply it                        |
| `ready-for-human`          | none                 | Not used                                        |
| `wontfix`                  | none                 | Close the issue as "not planned"                |

When a skill mentions a role (e.g. "apply the AFK-ready triage label"), use the corresponding
label string from this table. Where the table says "none", apply no label. A `wontfix` issue is
closed with `gh issue close <N> --reason "not planned"`.
