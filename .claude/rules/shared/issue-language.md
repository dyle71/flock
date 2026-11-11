# Issue language

Every issue an agent creates in the issue tracker — GitLab or GitHub, and whether the agent is
Claude, OpenCode, or any other — is written in **US-American English**. This holds no matter
which language the user speaks in the conversation.

The whole issue is English: the title, the description, every mandatory section
([work-issues.md](./work-issues.md)), the acceptance tests, and any comment the agent posts on
the issue afterward. If the user describes the work in German or any other language, translate
it — the issue is still authored in English.

- **US-American spelling**, never British: `-ize` / `-ization`, `color`, `behavior`, `center`,
  `gray`. This is the same spelling rule [code-language.md](./code-language.md) applies to code
  and repository documentation, extended here to the issue tracker.
- **Labels** stay exactly as [issue-labels.md](./issue-labels.md) prescribes — unaffected by
  this rule.

There are no exceptions. The conversation language never leaks into an issue.
