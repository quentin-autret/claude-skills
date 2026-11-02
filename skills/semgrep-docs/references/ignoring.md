# Ignoring files, folders and code

## Contents

- Default ignore list
- .semgrepignore
- Precedence and CLI filters
- Git interaction
- Inline nosemgrep
- Per-rule paths
- Platform path ignores
- Old patterns

Abbreviations used below: IGN = https://docs.semgrep.dev/ignoring-files-folders-code , V2 = https://docs.semgrep.dev/semgrepignore-v2-reference

## Default ignore list

Without any `.semgrepignore`, Semgrep skips (IGN):

```text
.git .svn .hg _darcs CVS
build/ vendor/ dist/ *.min.js .env/ .tox/
node_modules/ .npm/ .yarn/ .venv/ _opam/ _build/ _cargo/
test/ tests/ testsuite/ *_test.go
```

- `test/`, `tests/` and `vendor/` are not scanned by default. (IGN)
- The v2 reference lists a slightly different default set that also contains `.semgrep` and `.semgrep_logs/`. (V2)
- Out of scope regardless of ignores: files over 1 MB (`--max-target-bytes`), binary files, unknown extensions. (IGN)

## .semgrepignore

- Your own `.semgrepignore` replaces the default list, it does not extend it. Copy the defaults you want to keep. (IGN) https://docs.semgrep.dev/deployment/oss-deployment
- To scan tests, create a root `.semgrepignore` without the test entries. To scan everything, create an empty one (only `.git` stays ignored). (IGN)
- Syntax follows `.gitignore`. v2 is the default and only behavior since 1.117.0: several `.semgrepignore` files, the project's `.gitignore` files, and negation with `!` work. (V2) IGN still lists "character range" patterns as unsupported, while V2 lists them as unsupported only for legacy v1, so do not rely on them. https://docs.semgrep.dev/ignoring-files-folders-code
- Place it at the Git project root and run `semgrep` from there. Without Git, put it in the folder passed to `semgrep scan` (`foo/.semgrepignore` for `semgrep scan foo/`). https://docs.semgrep.dev/kb/semgrep-code/semgrepignore-ignored
- `:include .gitignore` pulls in another file's patterns (path relative to the including file, no nested includes). `\:` escapes a leading colon. (V2)
- `a/` floats (matches at any depth). `/a` and `a/b` are anchored to the file's directory. (V2)
- Commit `.semgrepignore` to Git. (IGN)
- Secrets scans ignore both the default list and every `.semgrepignore`. Use `--exclude` or the platform's Secrets path ignores. (IGN)
- GitLab's `SAST_EXCLUDED_PATHS` may not be honored by Semgrep: use `.semgrepignore` or the platform Path ignores instead. (IGN) https://docs.semgrep.dev/ignoring-files-folders-code GitLab's own Semgrep SAST analyzer does not read `.semgrepignore`. https://docs.semgrep.dev/deployment/oss-deployment

## Precedence and CLI filters

- Level 1 is `--exclude` / `--include`. Level 2 is the default list, `.gitignore` and `.semgrepignore`. A path excluded on the command line stays excluded even if `.semgrepignore` has `!path`. In 1 folder, `.gitignore` is read before `.semgrepignore`. (V2)
- `--exclude` and `--include` take gitignore globs and repeat. `--exclude=tests` drops `tests/foo.py` and `a/b/tests/c/foo.py`. `--include` cannot bring back a file ignored by `.gitignore` or `.semgrepignore`. https://docs.semgrep.dev/cli-reference (IGN)
- Symbolic links other than the scan roots are skipped, and so are Git submodules. (V2)
- `--x-ls` and `--x-ls-long` list the selected targets and why others were skipped (internal flags). `--project-root` forces the root. (V2)

## Git interaction

- A file listed in `.gitignore` but committed is still scanned. Putting `.gitignore` in `.semgrepignore` does not change that. List such files in `.semgrepignore` directly. (IGN)
- `--no-git-ignore` also scans gitignored untracked files and submodules. It has no effect outside a Git repository. https://docs.semgrep.dev/cli-reference (IGN)

## Inline nosemgrep

```python
bad_func1()  # nosemgrep
# nosemgrep: rule-id-1, rule-id-2
bad_func2()
```

- The comment goes on the first line of the match or on the line before. Name rules after a colon, comma separated. (IGN)
- Reference rule ids with their namespace. For rules stored in a `configs` subdirectory the id is `configs.rule-id-3`. (IGN)
- On the AppSec Platform the finding still exists and is set to Ignored automatically. The organization setting that allows inline comments must be on. (IGN)
- `--disable-nosem` negates `--enable-nosem`, which is on by default and suppresses findings on lines with a `nosem` comment. https://docs.semgrep.dev/cli-reference

## Per-rule paths

- Scope a single rule with its `paths.include` / `paths.exclude` key instead of `.semgrepignore`, which affects every rule. Its globs are relative to the project root. (IGN) https://docs.semgrep.dev/writing-rules/rule-syntax

## Platform path ignores

- `semgrep ci` also applies the path ignores set on the platform (project and organization, additive). `semgrep scan` only reads `.semgrepignore`. (IGN)
- Platform ignores cannot override the CLI defaults. To scan a default-ignored folder, you need a local `.semgrepignore` and a configured path ignores box. (IGN)

## Old patterns

| Old | Current | Source |
| :-- | :-- | :-- |
| `# nosem` | `# nosemgrep` | (IGN) |
| Semgrepignore v1 (1 file in the cwd, no `!`, no `[a-z]`, unsupported patterns silently dropped) | v2, the only behavior since 1.117.0 | (V2) https://docs.semgrep.dev/kb/semgrep-code/semgrepignore-ignored |
| `--semgrepignore-v2` flag | No flag needed | https://docs.semgrep.dev/cli-reference |
