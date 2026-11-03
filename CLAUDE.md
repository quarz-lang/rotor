# Working on rotor

A cog of Quarz (github.com/quarz-lang/quarz): its rules apply here.
Code, comments, commit messages and the docs are in English.

## Layout

- `Cog.toml` (`module = "Rotor"`), `src/` (declarations only),
  `test/*_test.qz` (`require "test"`, Test::Case), README.md, LICENSE.
- Depends on pg (`PG`), through `[patch]` on a checkout of it at
  `../quarz-pg` while both are developed together.
- The schema is generated at compile time by `static def`s (Quarz's
  docs/14); queries are `Quoted` blocks translated to SQL.

## Before committing

```sh
quarz check
quarz test              # interpreted
quarz test --release    # compiled
quarz test --doc        # the doc comments' examples
quarz doc --check
```

Use a `quarz` built from Quarz's main (`cargo build --release` there).
The tests that need a server (test/repo_test.qz) use pg's `PG::Testing`:
`PG_TEST_URL` when it is set (CI's `postgres:16` service sets it), else a
throwaway cluster; the others need none.

- Work on a branch and open a pull request; stage files by explicit path.
- Review every commit with a fresh subagent given the diff, and fix what
  it finds that needs no decision.
- Never print secrets (passwords, tokens).
- Do not put a `Claude-Session:` line in commit messages, nor a Claude
  session URL in pull request descriptions.
