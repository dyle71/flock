# Language

The language in the code base is US English without exceptions. Use US English for:

- Type names
- Variable names
- Method/function names
- All code comments — inline, block, and file-header descriptions
- Commit messages (outside this rule set but follow the same convention)
- All Markdown documentation — `CLAUDE.md`, `README.md`, the files under
  `.claude/rules/`, and everything under `docs/`. Two exemptions: localized user-facing content
  shipped as Markdown (`public/legal/**/*.de.md`) is copy, not documentation; and vendored
  third-party skills under `.claude/skills/` are not ours to rewrite.

**This applies regardless of what language is used in the conversation with the agent.**
If the user writes in German or any other language, all code and comments produced
must still be written in US English. There are no exceptions.

## US-American spelling

Use American spelling, never British:

- `-ize` / `-ization`, not `-ise` / `-isation`
- `-or`, not `-our`: `color`, `behavior`
- `-er`, not `-re`: `center`
- `gray`, not `grey`

This binds every identifier, comment, and string literal — and every Markdown document in the
repository. Prose in `*.md` files is held to the same spelling as code: write `capitalization`,
`behavior`, `organize`, `center`, never `capitalisation`, `behaviour`, `organise`, `centre`.

One thing is exempt. **Quoted external taxonomies** keep the spelling of their source. Seeded
reference data that reproduces a published classification — a standard's titles, an official
register's names — is a quotation, and americanizing it would make it diverge from the
published original.

The noun **`cancellation` keeps its double `l` in American English** — it is not a British
spelling and must never be "corrected" to `cancelation`. Only the verb forms follow the rule:
`canceled`, `canceling`.

Note the boundary: it governs **code and repository documentation**. User-facing copy follows
the project's own copy and formatting rules, which may well demand non-American date, time,
decimal, and unit formats even in English.
