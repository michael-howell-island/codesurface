# Startup Progress Reporting Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers-extended-cc:executing-plans to implement this plan task-by-task.

**Goal:** Stream concise per-file progress lines to stderr during `_index_full` so users can see indexing progress in real time.

**Architecture:** Pre-scan counts total matching files in one fast `os.walk` pass, then a throttled `on_progress` callback is passed down through `parse_directory` in each parser and called after each file. `_index_full` in `server.py` owns the closure, throttle logic, and stderr output.

**Tech Stack:** Python stdlib only (`os`, `time`, `sys`). No new dependencies.

---

### Task 1: Add `on_progress` to `BaseParser.parse_directory`

**Files:**
- Modify: `src/codesurface/parsers/base.py:28-60`
- Test: `tests/test_parsers.py`

**Step 1: Write the failing test**

Add to `tests/test_parsers.py`:

```python
def test_on_progress_called_per_file(ts_project):
    """on_progress is called once per successfully parsed file."""
    parser = TypeScriptParser()
    visited = []
    parser.parse_directory(ts_project, on_progress=lambda f: visited.append(f))
    # ts_project has service.ts, gen.ts, and a worktree service.ts (3 .ts files total without filter)
    assert len(visited) == 3
    assert all(isinstance(f, Path) for f in visited)


def test_on_progress_none_is_default(ts_project):
    """Omitting on_progress works exactly as before."""
    parser = TypeScriptParser()
    records = parser.parse_directory(ts_project)
    assert len(records) > 0
```

**Step 2: Run to verify it fails**

```bash
cd /Users/howell/code/codesurface
.venv/bin/pytest tests/test_parsers.py::test_on_progress_called_per_file -v
```

Expected: `FAILED` — `parse_directory() got an unexpected keyword argument 'on_progress'`

**Step 3: Update `base.py`**

Change the signature and body of `parse_directory`:

```python
def parse_directory(
    self, directory: Path, path_filter: "PathFilter | None" = None,
    on_progress: "Callable[[Path], None] | None" = None,
) -> list[dict]:
    """Recursively parse all matching files under *directory*."""
    exts = tuple(self.file_extensions)
    records = []

    for root, dirs, files in os.walk(directory):
        root_path = Path(root)

        if path_filter is not None:
            dirs[:] = [
                d for d in dirs
                if not path_filter.is_dir_excluded(root_path / d)
            ]

        for filename in files:
            if not filename.endswith(exts):
                continue
            f = root_path / filename
            if path_filter is not None and path_filter.is_file_excluded(f):
                continue
            try:
                records.extend(self.parse_file(f, directory))
                if on_progress is not None:
                    on_progress(f)
            except Exception:
                continue

    return records
```

Also add `Callable` to the imports at the top of `base.py`:

```python
from typing import TYPE_CHECKING, Callable
```

**Step 4: Run to verify it passes**

```bash
.venv/bin/pytest tests/test_parsers.py::test_on_progress_called_per_file tests/test_parsers.py::test_on_progress_none_is_default -v
```

Expected: both `PASSED`

**Step 5: Commit**

```bash
git add src/codesurface/parsers/base.py tests/test_parsers.py
git commit -m "feat: add on_progress callback to BaseParser.parse_directory"
```

---

### Task 2: Forward `on_progress` in subclass overrides

Four parsers override `parse_directory` with their own `os.walk` loops: `typescript.py`, `python_parser.py`, `go.py`, `java.py`. Each needs the same parameter added and called after `parse_file`.

**Files:**
- Modify: `src/codesurface/parsers/typescript.py:157-~190`
- Modify: `src/codesurface/parsers/python_parser.py:81-~115`
- Modify: `src/codesurface/parsers/go.py:149-176`
- Modify: `src/codesurface/parsers/java.py:139-167`
- Test: `tests/test_parsers.py`

**Step 1: Write the failing tests**

Add to `tests/test_parsers.py`:

```python
from codesurface.parsers.python_parser import PythonParser
from codesurface.parsers.go import GoParser
from codesurface.parsers.java import JavaParser


@pytest.fixture
def py_project(tmp_path):
    (tmp_path / "mod.py").write_text("def hello(): pass\n")
    return tmp_path


def test_typescript_on_progress(ts_project):
    parser = TypeScriptParser()
    visited = []
    parser.parse_directory(ts_project, on_progress=lambda f: visited.append(f))
    assert len(visited) >= 1


def test_python_on_progress(py_project):
    parser = PythonParser()
    visited = []
    parser.parse_directory(py_project, on_progress=lambda f: visited.append(f))
    assert len(visited) == 1


def test_go_on_progress(tmp_path):
    (tmp_path / "main.go").write_text("package main\nfunc Hello() {}\n")
    parser = GoParser()
    visited = []
    parser.parse_directory(tmp_path, on_progress=lambda f: visited.append(f))
    assert len(visited) == 1


def test_java_on_progress(tmp_path):
    (tmp_path / "Foo.java").write_text("public class Foo { public void bar() {} }\n")
    parser = JavaParser()
    visited = []
    parser.parse_directory(tmp_path, on_progress=lambda f: visited.append(f))
    assert len(visited) == 1
```

**Step 2: Run to verify they fail**

```bash
.venv/bin/pytest tests/test_parsers.py::test_typescript_on_progress tests/test_parsers.py::test_python_on_progress tests/test_parsers.py::test_go_on_progress tests/test_parsers.py::test_java_on_progress -v
```

Expected: all `FAILED` — `parse_directory() got an unexpected keyword argument 'on_progress'`

**Step 3: Update each subclass**

The pattern is identical for all four. For each parser, change the signature and add the callback call after `records.extend(self.parse_file(...))`:

**Pattern to apply to each parser's `parse_directory`:**

```python
def parse_directory(
    self, directory: Path, path_filter: "PathFilter | None" = None,
    on_progress: "Callable[[Path], None] | None" = None,
) -> list[dict]:
    # ... existing walk logic unchanged ...
    for filename in files:
        # ... existing skip checks unchanged ...
        try:
            records.extend(self.parse_file(f, directory))
            if on_progress is not None:    # ← add this
                on_progress(f)             # ← add this
        except Exception ...:
            continue
    return records
```

Also add `Callable` to each parser's imports:

```python
from typing import TYPE_CHECKING, Callable
```

Apply the same pattern to all four: `typescript.py`, `python_parser.py`, `go.py`, `java.py`.

**Step 4: Run to verify they pass**

```bash
.venv/bin/pytest tests/test_parsers.py -v
```

Expected: all tests `PASSED`

**Step 5: Commit**

```bash
git add src/codesurface/parsers/typescript.py src/codesurface/parsers/python_parser.py src/codesurface/parsers/go.py src/codesurface/parsers/java.py tests/test_parsers.py
git commit -m "feat: forward on_progress callback in all parser subclass overrides"
```

---

### Task 3: Add `_count_files` helper and progress closure to `server.py`

**Files:**
- Modify: `src/codesurface/server.py:32-81`
- Test: `tests/test_server_progress.py` (new file)

**Step 1: Write the failing tests**

Create `tests/test_server_progress.py`:

```python
"""Tests for _count_files and _index_full progress output."""
import sys
from pathlib import Path
import pytest


def test_count_files_basic(tmp_path):
    """_count_files counts .py files, respects extension list."""
    from codesurface.server import _count_files
    from codesurface.parsers.python_parser import PythonParser

    (tmp_path / "a.py").write_text("x = 1")
    (tmp_path / "b.py").write_text("y = 2")
    (tmp_path / "c.txt").write_text("ignored")

    parsers = [PythonParser()]
    assert _count_files(tmp_path, parsers, path_filter=None) == 2


def test_count_files_prunes_excluded_dirs(tmp_path):
    """_count_files respects path_filter dir exclusions."""
    from codesurface.server import _count_files
    from codesurface.parsers.python_parser import PythonParser
    from codesurface.filters import PathFilter

    (tmp_path / "src").mkdir()
    (tmp_path / "src" / "a.py").write_text("x = 1")
    wt = tmp_path / ".worktrees" / "pr-1"
    wt.mkdir(parents=True)
    (wt / ".git").write_text("gitdir: /repo/.git/worktrees/pr-1\n")
    (wt / "b.py").write_text("y = 2")

    pf = PathFilter(tmp_path)
    parsers = [PythonParser()]
    assert _count_files(tmp_path, parsers, path_filter=pf) == 1


def test_index_full_emits_progress_to_stderr(tmp_path, capsys):
    """_index_full prints at least one progress line and a done line to stderr."""
    from codesurface import server
    from codesurface.filters import PathFilter

    # Create enough .py files to trigger at least the initial 0% line
    for i in range(5):
        (tmp_path / f"m{i}.py").write_text(f"def f{i}(): pass\n")

    server._conn = None
    server._project_path = tmp_path
    server._path_filter = PathFilter(tmp_path)

    server._index_full(tmp_path)

    captured = capsys.readouterr()
    assert "[codesurface]" in captured.err
    assert "done:" in captured.err
```

**Step 2: Run to verify they fail**

```bash
.venv/bin/pytest tests/test_server_progress.py -v
```

Expected: `FAILED` — `cannot import name '_count_files'`

**Step 3: Implement `_count_files` and update `_index_full` in `server.py`**

Add `_count_files` just before `_index_full`:

```python
def _count_files(
    project_path: Path,
    parsers: list,
    path_filter: "PathFilter | None",
) -> int:
    """Quick pre-scan: count source files that will be parsed."""
    extensions = set()
    for p in parsers:
        extensions.update(p.file_extensions)
    exts = tuple(extensions)
    total = 0
    for root, dirs, files in os.walk(project_path):
        root_path = Path(root)
        if path_filter is not None:
            dirs[:] = [d for d in dirs if not path_filter.is_dir_excluded(root_path / d)]
        for filename in files:
            if filename.endswith(exts):
                total += 1
    return total
```

Update `_index_full` to add progress reporting:

```python
def _index_full(project_path: Path, language: str | None = None) -> str:
    """Full parse + rebuild. Used on startup."""
    global _conn, _file_mtimes
    t0 = time.perf_counter()

    if language:
        parsers = [get_parser(language)]
    else:
        parsers = get_parsers_for_project(project_path)

    if not parsers:
        return "No supported source files detected in project directory."

    total = _count_files(project_path, parsers, _path_filter)
    print(f"[codesurface] scanning {total:,} files...", file=sys.stderr, flush=True)

    parsed = [0]
    last_pct = [-1.0]
    last_time = [time.perf_counter()]

    def on_progress(f: Path) -> None:
        parsed[0] += 1
        now = time.perf_counter()
        pct = parsed[0] / max(total, 1)
        elapsed = now - t0
        if pct - last_pct[0] >= 0.05 or now - last_time[0] >= 3.0:
            print(
                f"[codesurface] indexing: {pct:3.0%} ({parsed[0]:>6,} / {total:,})  {elapsed:.1f}s",
                file=sys.stderr,
                flush=True,
            )
            last_pct[0] = pct
            last_time[0] = now

    # Emit the 0% line immediately
    on_progress.__wrapped_call_count = 0  # unused, just emit 0% manually
    print(
        f"[codesurface] indexing:   0% ({0:>6,} / {total:,})",
        file=sys.stderr,
        flush=True,
    )
    last_pct[0] = 0.0

    records = []
    for parser in parsers:
        records.extend(
            parser.parse_directory(project_path, path_filter=_path_filter, on_progress=on_progress)
        )
    parse_time = time.perf_counter() - t0

    t1 = time.perf_counter()
    _conn = db.create_memory_db(records)
    db_time = time.perf_counter() - t1

    # Snapshot mtimes (unchanged)
    extensions = set()
    for parser in parsers:
        extensions.update(parser.file_extensions)
    exts = tuple(extensions)

    _file_mtimes = {}
    for root, dirs, files in os.walk(project_path):
        root_path = Path(root)
        if _path_filter is not None:
            dirs[:] = [d for d in dirs if not _path_filter.is_dir_excluded(root_path / d)]
        for filename in files:
            if not filename.endswith(exts):
                continue
            f = root_path / filename
            rel = str(f.relative_to(project_path)).replace("\\", "/")
            try:
                _file_mtimes[rel] = f.stat().st_mtime
            except OSError:
                pass

    stats = db.get_stats(_conn)
    langs = ", ".join(type(p).__name__.replace("Parser", "") for p in parsers)
    summary = (
        f"[codesurface] done: {stats['total']:,} records from {stats.get('files', 0):,} files "
        f"({langs}) in {parse_time + db_time:.2f}s"
    )
    print(summary, file=sys.stderr, flush=True)
    return summary
```

Note: the `on_progress.__wrapped_call_count` line is a no-op artifact — remove it and just emit the 0% line directly as shown.

**Step 4: Run to verify tests pass**

```bash
.venv/bin/pytest tests/test_server_progress.py -v
```

Expected: all `PASSED`

**Step 5: Run full test suite**

```bash
.venv/bin/pytest -v
```

Expected: all tests pass

**Step 6: Commit**

```bash
git add src/codesurface/server.py tests/test_server_progress.py
git commit -m "feat: stream indexing progress to stderr with file count and percentage"
```

---

### Task 4: Smoke test end-to-end

**Step 1: Restart codesurface with a real project**

```bash
cd /Users/howell/code/codesurface
.venv/bin/python -m codesurface.server --project /Users/howell/work/cloud 2>&1 | head -20
```

Expected output shape:
```
[codesurface] scanning 1,234 files...
[codesurface] indexing:   0% (     0 / 1,234)
[codesurface] indexing:  10% (   123 / 1,234)  4.2s
...
[codesurface] done: 8,412 records from 1,234 files (CSharp) in 22.1s
```

**Step 2: Verify no regressions on a small project**

```bash
.venv/bin/python -m codesurface.server --project /Users/howell/code/codesurface 2>&1 | head -10
```

Expected: progress lines followed by `done:` line, no tracebacks.

**Step 3: Commit if any fixups needed, otherwise done**
