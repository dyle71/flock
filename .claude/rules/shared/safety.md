# Safety and scope

- **Never** run destructive commands: `rm -rf`, `sudo`, force-push, mass deletes. Do not attempt
  them.
- **Never** read or print secrets: `.env` and `.env.*` are off-limits (except `.env.example` /
  `.env.sample`, and any env file a project rule declares checked in). Do not echo tokens, keys,
  or credentials.
- Stay **inside the project directory**. Do not touch files outside the repo.
- **Never commit, stage, push, or create, switch or delete a branch.** Version control is the
  user's job — see [version-control.md](./version-control.md). That rule is absolute: there is
  no "confirm first" path for `git commit`, `git add`, `git push`, `git switch` or
  `git checkout -b`.
- **Confirm before:** deleting files, editing CI/deploy configs, or running database migrations
  against anything that isn't a throwaway dev DB.
- Do not hand-edit generated files or lockfiles.
