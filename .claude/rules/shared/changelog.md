# Changelog

Loaded unconditionally, never path-scoped: every issue that changes something a user or an
operator would notice ends with a changelog entry, whichever files it touched.

The file is `CHANGELOG.md` in the repository root. Its format is
[Keep a Changelog 1.0.0](https://keepachangelog.com/en/1.0.0/).

## The version heading is the project's current version

The version declared by the project's manifest — whichever file the project keeps its version
in — names the section an entry belongs to. Read it before writing anything.

- **The section exists** — append the entry to it, and set the section's date to today.
- **The section does not exist** — create it at the top of the file, directly under the
  introduction, above the previous version's section. Heading format:
  `## [<version>] - <YYYY-MM-DD>`, dated the day it is created.

### An unreleased section is dated by its most recent entry

A version still in development — one that carries no tag in the [Links footer](#the-links-footer)
— is dated by the last thing that happened to it, not by the day somebody first opened it.
**Every time an entry is added to such a section, its heading date is set to the current date.**
A section opened on 2026-09-09 and appended to on 2026-09-10 reads:

```markdown
## [0.5.0] - 2026-09-10
```

Read the current date before writing the entry rather than assuming the section's date is still
right; a section usually outlives the day it was opened.

**Tagging freezes the date.** From the moment a version appears in the Links footer, its date
records when that release happened and nothing may move it again.

Never renumber, reorder, or rewrite a section that has been released. Older entries are history.
That protects their **content**, not their formatting: reflowing an old entry to the 100-column
width below is fine, as long as not one word changes. The heading date of the unreleased section
is the single exception, and it moves forward only — never back.

## Appending an entry

Within a version section, entries live under the Keep a Changelog category headings, in this
order, and a category heading is written only when it has at least one entry:

| Heading          | For                                |
| ---------------- | ---------------------------------- |
| `### Added`      | New features.                      |
| `### Changed`    | Changes to existing functionality. |
| `### Deprecated` | Soon-to-be-removed features.       |
| `### Fixed`      | Bug fixes.                         |
| `### Removed`    | Features removed in this release.  |
| `### Security`   | Vulnerabilities fixed.             |

An entry is **a single list item, appended to the end of its category list**. Wrap it at
**100 columns**; continuation lines are indented by two spaces so the whole entry stays one
bullet:

```markdown
- Short, precise statement of the change, wrapped at 100 columns with a two-space indent on
  every continuation line ([#42](https://example.com/group/project/-/issues/42))
```

- **One sentence, no trailing period**, written in the imperative-free descriptive past
  ("Added the export dialog" is redundant under `### Added` — write "Export dialog for wheel
  participants").
- **Always reference the issue** as a Markdown link whose text is `#<id>` and whose URL is
  that issue's URL in the project's issue tracker. An entry without an issue link is
  incomplete; if the work has no issue, ask the user.
- **US-American English**, per [code-language.md](./code-language.md).
- **Dates are ISO 8601** (`2026-09-09`) in this file. A date format the project prescribes
  elsewhere governs user-facing copy, not repository documentation, and Keep a Changelog
  mandates ISO 8601 here.
- **Never a CUID, never personal data, never a secret** in an entry.

## When to write one

The test is **size, not visibility**. Write an entry for every substantial change, whether or
not somebody outside the repository could notice it: a feature, a behavior change, a bug fix,
a security fix, a removed or deprecated capability, a new environment variable, a new build or
run script — and equally a refactor, a restructuring, a dependency upgrade, or a schema
migration. **A refactor is a large change by its nature and belongs in the changelog**, even
when it is behavior-preserving by design; record it under `### Changed`.

Do not write one for changes too small to be worth a reader's time: a formatting pass, a typo,
a reworded comment, a test-only change, or an edit to the agent instruction files themselves.

When in doubt, write the entry. A line that turns out to be uninteresting costs a reader two
seconds; a missing one costs whoever is reconstructing when something changed far more.

Adding the entry is part of the work, not a follow-up: it happens in the same branch as the
change it describes. Never commit it yourself — [version-control.md](./version-control.md)
still binds.

## The Links footer

The file ends with a `## Links` section: the project's repository, its long-lived branches,
and one entry per released tag. It is reference material for a reader who arrives at the
changelog and needs to get to the code.

**When a version is tagged, add that tag to the Links footer**, in the same step that creates
the tag, newest first. A version section carrying no tag in the footer is still in
development. Never invent one of these URLs — read the branch and tag names from the
repository host first, the way [agent-conduct.md](./agent-conduct.md) demands.
