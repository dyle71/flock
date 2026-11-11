# Work Issues

Issues are the contract between whoever specifies the work and whoever implements it —
increasingly an AI agent rather than a person. **Write every issue so that a weak, literal,
low-capability AI can implement it end to end without asking a follow-up question and without
inventing anything.** That is the single quality bar. If an issue only makes sense to someone
who already sat in the meeting, it is not finished.

## Every issue must be SMART

| Letter         | What it demands of the issue                                                                               |
| :------------- | :--------------------------------------------------------------------------------------------------------- |
| **Specific**   | Name the exact files, routes, models, functions, and translation keys involved. No "improve the export".   |
| **Measurable** | State the acceptance tests. Done is "these assertions pass", never "it works".                             |
| **Achievable** | One issue is one deliverable slice, implementable in a single branch. Split anything larger.               |
| **Relevant**   | Tie the issue to the user-visible outcome or the legal/technical obligation it serves (e.g. GDPR Art. 20). |
| **Time-bound** | Carry a milestone or an explicit "blocks / blocked by" chain, so scheduling is never implicit.             |

## Every issue is acceptance-test driven

Write the acceptance tests **before** the implementation section, and write them as the
executable checks they will become. The implementer's job is to make them pass — nothing more.

- Express each criterion as **Given / When / Then**, with concrete values, not placeholders.
- Name the test tier each test belongs to (unit, integration, end-to-end) and the file it
  will live in.
- Cover the failure paths too: unauthorized caller, missing record, invalid input.
- An issue with no acceptance test is not ready to be worked on. Ask for one.

## Mandatory issue sections

Every issue body carries these four sections, in this order, with these headings:

```markdown
## Instructions

The ordered, imperative steps to perform. Each step names its file and its change.
Written for someone with no prior knowledge of this codebase.

## Context

Why this exists, what already exists, and where. Link the code that will be touched or
imitated, the related issues, and the domain rules a newcomer cannot infer.

## Constraints

What must NOT change, and which project rules bind this work: the applicable rules from
this repository's instruction files, plus any performance, schema, or
backwards-compatibility limit.

## Acceptance Tests

Given / When / Then, per the section above.
```

## Writing for a weak implementer

Assume the implementer cannot infer, cannot browse widely, and will take every sentence
literally.

- **Spell out the obvious.** Name every file that must change, even when a convention would
  imply it. Do not rely on the reader remembering a rule from elsewhere.
- **Give exact identifiers.** Full paths, exact function names, exact permission action
  strings, exact enum members. Never "the relevant hook".
- **One instruction per step.** No compound steps joined by "and then also".
- **Show, don't describe.** Paste the code snippet, the expected data shape, the exact copy
  in every locale. A weak model reproduces examples reliably and invents badly.
- **State the finish line.** The last instruction is always which commands must pass and
  which test files must be green.
- **No open questions in the body.** If the issue still contains a question, it is a
  `type: suggestion`, not an implementable issue ([issue-labels.md](./issue-labels.md)).

## Starting work on an issue

When an agent starts working on an issue, it first assigns the issue to the authenticated
`gh` user, so that everyone — and every other agent session — sees it is taken:

```bash
gh issue edit <N> --add-assignee @me
```

- Always the additive form. Never remove or replace an existing assignee.
- Never reopen a closed issue for this. Ask the user instead.
- Change nothing else on the issue: no labels, no milestone, no comment.
