# Codesurface: File Filtering & Worktree Support

**Date:** 2026-03-29
**Status:** Approved

## Problem

Pointing codesurface at a repo root that contains `.worktrees/` causes it to index
every worktree (full repo copies), multiplying work N times and hanging on startup.
There's also no way to exclude generated files or scope queries to a subset of the
index.

## Goals

1. Never index worktrees or submodules by default
2. Let users exclude paths at index time (CLI + config file)
3. Let callers scope queries to a file path or glob at query time
4. Support per-worktree indexing from pr-manager

---

## Feature 1: Default Skip Rules

Built-in exclusions applied before any file is parsed or tracked in `_file_mtimes`.

**Rule 1 — `.worktrees` directory name**
Any directory named `.worktrees` is skipped unconditionally. Fast path, no filesystem
reads needed.

**Rule 2 — `.git` FILE detection**
Any subdirectory containing a `.git` FILE (not directory) is a detached checkout —
either a git worktree or a submodule. Skip it. Both produce a `.git` file; the
distinction (worktree refs `/worktrees/`, submodule refs `/modules/`) doesn't matter
because we want to skip both by default.

**`--include-submodules` flag**
Opt-in. When set, subdirs whose `.git` file references `/modules/` are re-included.
Worktrees (referencing `/worktrees/`) are always excluded regardless.

---

## Feature 2: User-Configurable Exclusions

### `.codesurfaceignore`
Gitignore-style file read from the project root. One glob per line. `#` comments
supported. Patterns are matched against relative file paths.

```
# .codesurfaceignore example
tests/**
**/generated/**
vendor/
src/legacy/
```

### `--exclude` CLI flag
Comma-separated globs. Merged with `.codesurfaceignore` patterns. Useful for
ad-hoc exclusions without editing the project.

```
uvx codesurface --project ~/work/apps --exclude "tests/**,**/*.generated.ts"
```

### `PathFilter` class
Both sources are compiled into a single `PathFilter` object used in `_index_full`
and `_index_incremental`. Excluded files are never added to `_file_mtimes`, so
incremental reindex ignores them too.

---

## Feature 3: Query-time File Scoping

New optional `file_path` parameter on `search`, `get_signature`, and `get_class`.

```
search("MergeService", file_path="src/services/")
get_class("IMergeService", file_path="src/services/MergeService.cs")
```

- **Prefix match** (no wildcards): `WHERE file_path LIKE 'src/services/%'`
- **Glob** (contains `*` or `?`): post-filter with `fnmatch` after DB query

`file_path` is already stored on every record — no schema changes needed.

---

## Feature 4: Per-Worktree Indexing (pr-manager side)

Codesurface already supports pointing `--project` at any directory. The pr-manager
`mcp-proxy-manager.js` already manages dynamic codegraph servers for review
worktrees (`codegraph-review-<pr>`). We add parallel `codesurface-review-<pr>`
servers using the existing `TEMPLATES.codesurface` builder.

Lifecycle mirrors codegraph review servers:
- Created when a review worktree is set up
- Torn down when the worktree is removed
- Registered as an SSE proxy at `/api/mcp-proxy/codesurface-review-<pr>/sse`

---

## Implementation Scope

All changes are in `~/code/codesurface` except Feature 4 which is in
`~/code/pr-manager/server/lib/mcp/mcp-proxy-manager.js` and
`~/code/pr-manager/server/routes/pr-graph.js`.

### Files changed in codesurface

| File | Changes |
|------|---------|
| `src/codesurface/server.py` | `--exclude`, `--include-submodules` flags; pass `PathFilter` to index functions; add `file_path` param to tools |
| `src/codesurface/filters.py` | New — `PathFilter` class, `.codesurfaceignore` reader, git-checkout detection |
| `src/codesurface/parsers/base.py` | Accept `PathFilter` in `parse_directory` |

### Files changed in pr-manager

| File | Changes |
|------|---------|
| `server/lib/mcp/mcp-proxy-manager.js` | Spin up `codesurface-review-<pr>` alongside `codegraph-review-<pr>` |
| `server/routes/pr-graph.js` | Start/stop codesurface review server in worktree setup/teardown |
