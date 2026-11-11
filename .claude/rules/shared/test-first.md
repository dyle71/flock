# Test first

Loaded unconditionally, never path-scoped: the order of the work is fixed before the first file
is touched.

Once the user has told the agent to implement an issue, the work runs in the phases below, in
this order. **The specification is the single source of truth.** The issue and its acceptance
tests are derived from it; where they disagree, the specification wins and the user decides.

The project's own rules name where its specification lives and which commands run its tests.

## Which issues

An issue that changes behavior — a feature, a bug fix — runs all phases.

An issue that changes no behavior — a refactoring, documentation, configuration — has no red
phase: every existing test stays green and unchanged ([Frozen tests](#frozen-tests)).

The agent states its classification at the start, before phase 1. When it is unclear, it asks.

## Phase 1 — check the acceptance tests against the specification

The specification of an issue is the set of sections the issue cites from the project's
specification, plus the decision records and the glossary it cites. The issue body is not a
source of truth. An issue that cites no specification is not checked against its own text: the
agent stops and asks.

Three checks, each one on every acceptance test of the issue:

1. **Derivable.** Every acceptance test follows from the specification. A test that asserts
   behavior the specification does not state is invented.
2. **Covered.** Every requirement of the cited sections that falls within the issue's scope is
   covered by at least one acceptance test.
3. **Exact.** Every concrete value in a test — a status code, a message, a number, a field —
   stands in the specification or follows from it necessarily.

A test follows **directly** when the specification states it. It follows **indirectly** when
the derivation runs through a decision record, the glossary or a project rule. Every
derivation names its source: file and section.

### The report

The agent always reports the result, also when everything matches. The report is written for a
reader who has read neither the specification, nor the issue, nor the code in this session, and
who must decide on its basis alone. A table with one line of source per test is an overview,
never the report. The report has these parts, in this order:

1. **The basis.** The files and sections that make up the specification of the issue. For
   each: whether the issue amends it, and whether that amendment is already in the repository.
2. **What exists today.** The current behavior the issue changes, with `file:line`, so the
   reader sees what each test will turn red against.
3. **The overview**, as a table:

   | #   | Acceptance test | Tier, file       | Verdict                            |
   | --- | --------------- | ---------------- | ---------------------------------- |
   | n   | `<test>`        | `<tier>, <file>` | direct / indirect / partial / none |

4. **One section per acceptance test**, numbered as in the overview, with:
   - **What it checks**: the Given, When and Then in full, with every concrete value.
   - **The source**: the sentence of the specification it derives from, quoted verbatim with
     file, section and line. An indirect derivation quotes every link of the chain the same
     way: the decision record, the glossary entry, the project rule.
   - **The derivation**: in a few sentences, why the test follows from the quoted text.
   - **Every concrete value**: each status code, error code, message, number, field and copy
     string the test asserts, each with its own source or "none".
   - **On a mismatch**: what is missing or differs, where the value the test uses actually
     comes from (the issue's instructions, the code, an env file), and what goes wrong in the
     implementation if it stays unresolved.
5. **Uncovered requirements.** Each quoted verbatim with its source, why no test covers it, and
   the test that would, written as Given / When / Then.
6. **The decisions needed.** One per open point, each with its context, the options, what each
   option changes in the specification, the issue and the tests, and the agent's
   recommendation with its reason.

**On any mismatch the agent stops** — a test without a source, an uncovered requirement, a
value that differs. It changes neither the issue nor the specification, and it does not start
the red phase until the user has decided.

## Phase 2 — red

The agent writes every acceptance test of the issue first, before any implementation.

- **Every new test fails on an assertion** that checks the specified behavior — never on an
  import, a missing symbol or a type error. A symbol the tests need but that does not exist yet
  gets a minimal stub (its signature, a body that throws "not implemented"). The stubs are
  implementation, not test, and are not frozen. The type check stays green.
- **A new test that passes at once stops the work.** Either the behavior exists already, or the
  test checks nothing. The agent reports it and never rewrites the test until it turns red.

At the end of the phase the agent shows the new tests and the red test output, and stops.

## Phase 3 — freeze

The user's "ok" freezes the new tests and releases the implementation. Nothing else does.

## Phase 4 — implement

The agent implements until every test is green: the new ones and every existing one, across
every test tier, each run seen in this session. The project's definition of done names the
commands.

While implementing, the agent may write additional tests of its own in a red-green cycle —
unit tests of the parts it builds. Each one is frozen the moment it is written.

For the acceptance tests of an issue, this rule outranks the vertical slicing of a TDD skill:
all of them are written before the implementation, then frozen.

## Frozen tests

Every test in the repository is frozen: the tests of earlier issues, the ones the red phase
wrote, the ones written during implementation, the shared fixtures and helpers, and the guard
tests together with their allowlists.

- **Adding** a new test case — also into an existing test file — and adding a new helper is
  allowed.
- **Changing or deleting** a test case, a fixture, a helper, a guard test or an allowlist entry
  needs the user's explicit "ok". The agent shows the diff and the reason — for an existing
  test that breaks because the specification changes the behavior, the section that says so —
  and waits. The "ok" covers exactly that diff.
- Mechanical changes count: a moved import, a renamed symbol, an adjusted fixture. The single
  exception is the formatter's own output, which changes no assertion.
- A red test is fixed in the implementation, never in the test.
