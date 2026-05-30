---
name: github-safe-upload
description: Use when preparing to upload, push, publish, or mirror a local repository to GitHub and the user wants to know what is safe to include, exclude, commit, or push.
---

# GitHub Safe Upload

## Core Rule

Do a read-only upload audit before staging, committing, pushing, or creating files remotely. Classify files as `upload`, `exclude`, or `fix first`; then explain the reasoning clearly.

## Audit Steps

1. Inspect repository state and remotes:

```bash
git status --short --untracked-files=all
git status --ignored --short --untracked-files=all
git remote -v
git ls-files
```

2. Inventory local files, excluding `.git` noise when useful:

```bash
find . -maxdepth 4 -path ./.git -prune -o -print
find . -maxdepth 5 \( -name '*.key' -o -name '*.pem' -o -name '*.crt' -o -name '*.p12' -o -name '*.pfx' -o -name '.env' -o -name '.env.*' -o -name '*secret*' -o -name '*token*' -o -name '*password*' \) -print
```

3. Search for likely secrets without printing secret contents unnecessarily:

```bash
rg -n "(password|passwd|secret|private[_-]?key|api[_-]?key|token|dsn|BEGIN|AGE-SECRET|C2_MYSQL_DSN)" .
```

If matches are likely real credentials, do not quote them back. Name the file and field type only.

4. Inspect archives and generated bundles before approving them:

```bash
file path/to/archive-or-binary
tar -tzf path/to/archive.tar.gz
tar -xOzf path/to/archive.tar.gz suspicious/path 2>/dev/null
```

Generated archives are `fix first` if they contain local configs, keys, `.env`, build outputs, IDE files, or nested worktrees.

5. Check ignore rules:

```bash
sed -n '1,220p' .gitignore
sed -n '1,120p' .git/info/exclude
git check-ignore -v path/to/suspicious/file
```

If `.gitignore` is locally excluded, call it out; it may need forced staging or `.git/info/exclude` cleanup.

## Classification Guide

`exclude`:
- `.git/`, `.claude/`, `.codex/`, `.idea/`, `.vscode/`, local worktrees, editor settings.
- `.env`, `.env.*`, real DSNs, tokens, passwords, API keys.
- Runtime keys such as `*.key`, `*.pem`, Age private keys, SSH keys, certificates.
- Local implant/client/server configs containing generated keys, real URLs, real callbacks, or machine-specific state.
- Build artifacts: `bin/`, `build/`, `dist/`, `tmp/`, `*.exe`, `*.dll`, `*.so`, `*.dylib`, `*.test`, coverage files.

`upload`:
- Source code, tests, `go.mod`, `go.sum`, docs, examples, templates.
- Example configs that contain only localhost, placeholders, or empty credential fields.
- Generated source files required to build the project, such as protobuf `.pb.go`, when they do not contain secrets.

`fix first`:
- Archives or embedded assets required by the build but currently bundling excluded files.
- `.gitignore` missing important ignore patterns.
- Files with placeholder-looking credentials that might actually be real for the user.
- Tracked secrets: recommend removal from future commits and rotation if already pushed.

## Response Format

Lead with the decision:

- `Safe to upload now`
- `Not safe yet`
- `Safe after these fixes`

Then provide short sections:

- `Do not upload`
- `Upload`
- `Fix before upload`
- `Commands I would use next`

Never stage, commit, push, or create/update GitHub files until the user has seen the classification and approved proceeding.

## Common Mistakes

- Do not trust `.gitignore` alone; check `git status --ignored` and tracked files.
- Do not approve tarballs, zip files, or generated source snapshots without listing their contents.
- Do not print private keys, tokens, or full credentials into chat.
- Do not upload tool worktrees or duplicated local scratch copies.
- Do not treat public keys as always harmless; upload only when they are intended project material, not a runtime pair tied to a private key.
