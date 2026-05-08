<!-- mcp-name: io.github.Codeturion/codesurface -->

# codesurface

[![PyPI Version](https://img.shields.io/pypi/v/codesurface.svg)](https://pypi.org/project/codesurface/)
[![PyPI Downloads](https://img.shields.io/pypi/dm/codesurface.svg)](https://pypi.org/project/codesurface/)
[![MCP Registry](https://img.shields.io/badge/MCP-Registry-green)](https://registry.modelcontextprotocol.io/?q=codesurface)
[![GitHub Stars](https://img.shields.io/github/stars/Codeturion/codesurface)](https://github.com/Codeturion/codesurface)
[![GitHub Last Commit](https://img.shields.io/github/last-commit/Codeturion/codesurface)](https://github.com/Codeturion/codesurface)
[![Languages](https://img.shields.io/badge/languages-C%23%20%7C%20TS%20%7C%20Java%20%7C%20Go%20%7C%20Python-blueviolet)](https://github.com/Codeturion/codesurface)
[![License: PolyForm Noncommercial](https://img.shields.io/badge/License-PolyForm%20Noncommercial%201.0.0-blue.svg)](https://polyformproject.org/licenses/noncommercial/1.0.0/)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![Blog Post](https://img.shields.io/badge/Blog-Benchmark%20Write--up-blue)](https://www.codeturion.me/blog/reducing-llm-agent-hallucinations-through-constrained-api-retrieval)

**MCP server that indexes your codebase's public API at startup and serves it via compact tool responses — saving tokens vs reading source files.**

Parses source files, extracts public classes/methods/properties/fields/events, and serves them through 5 MCP tools. Works with Claude Code, Cursor, Windsurf, or any MCP-compatible AI tool.

**Supported languages:** C# (`.cs`), Go (`.go`), Java (`.java`), Python (`.py`), TypeScript/TSX (`.ts`, `.tsx`)

## Quick Start

Add to your `.mcp.json`:

```json
{
  "mcpServers": {
    "codesurface": {
      "command": "uvx",
      "args": ["codesurface", "--project", "/path/to/your/src"]
    }
  }
}
```

Point `--project` at any directory containing supported source files — a Unity `Assets/Scripts` folder, a Spring Boot project, a .NET `src/` tree, a Node.js/React project, a Python package, etc. Languages are auto-detected.

Restart your AI tool and ask: *"What methods does MyService have?"*

## CLAUDE.md Snippet

Add this to your project's `CLAUDE.md` (or equivalent instructions file). **This step is critical for adoption.** Without it, the AI has the tools but defaults to Grep/Read instead.

````markdown
## codesurface — symbol discovery (use BEFORE Grep/Read)

**When you need to find a class, interface, method, or type — use codesurface search, not Grep.**
A codesurface search returns ranked results with full signatures, parameter types, return types,
and exact file:line locations. One call gives you a working mental model of a class without
opening the file. Grep returns raw file content that costs 5-12x more tokens.

**Workflow — follow this order:**

1. `search("keyword")` → find the symbol (returns file path + line number)
2. `get_class("ClassName")` → see all members if you need the full API surface
3. `get_signature("methodName")` → get exact params + return type
4. `Read(file, offset=line, limit=15)` → only now read the implementation if needed

For pattern-based discovery, use regex: `search("I.*Service$", regex=True)`

**Never skip to Grep/Read when codesurface can answer the question.** Fall back to Grep only for:
string literals, configuration values, usage patterns across files, or when codesurface returns no results.

This applies to you AND any subagents you spawn.
````

## Tools

| Tool | Purpose | Example |
|------|---------|---------|
| `search` | Find APIs by keyword or regex | `search("MergeService")`, `search("I.*Service$", regex=True)` |
| `get_signature` | Exact signature by name or FQN | "TryMerge", "CampGame.Services.IMergeService.TryMerge" |
| `get_class` | Full class reference card — all public members | "BlastBoardModel" → all methods/fields/properties |
| `get_stats` | Overview of indexed codebase | File count, record counts, namespace breakdown |
| `reindex` | Incremental index update (mtime-based) | Only re-parses changed/new/deleted files. Also runs automatically on query misses |

### Regex Search

Pass `regex=True` to treat the query as a Python regex pattern, matched case-insensitively against `fqn`, `class_name`, `member_name`, and `signature`:

```
search("I.*Service$", regex=True)                    # all interfaces ending in Service
search("get(User|Account)", regex=True, member_type="method")  # specific getter methods
search("constructor.*ILog", regex=True)               # classes injecting ILog
```

All standard filters (`member_type`, `file_path`, `include_tests`, `n_results`) work with regex mode.

## Tested On

| Project | Language | Files | Records | Time |
|---------|----------|-------|---------|------|
| [vscode](https://github.com/microsoft/vscode) | TypeScript | 6,611 | 88,293 | 9.3s |
| [Paper](https://github.com/PaperMC/Paper) | Java | 2,909 | 33,973 | 2.3s |
| [client-go](https://github.com/kubernetes/client-go) | Go | 219 | 2,760 | 0.4s |
| [langchain](https://github.com/langchain-ai/langchain) | Python | 1,880 | 12,418 | 1.1s |
| [pydantic](https://github.com/pydantic/pydantic) | Python | 365 | 9,648 | 0.3s |
| [guava](https://github.com/google/guava) | Java | 891 | 8,377 | 2.4s |
| [immich](https://github.com/immich-app/immich) | TypeScript | 919 | 7,957 | 0.6s |
| [fastapi](https://github.com/tiangolo/fastapi) | Python | 881 | 5,713 | 0.5s |
| [ant-design](https://github.com/ant-design/ant-design) | TypeScript | 2,947 | 5,452 | 0.9s |
| [dify](https://github.com/langgenius/dify) | TypeScript | 4,903 | 5,038 | 1.9s |
| [crawlee-python](https://github.com/apify/crawlee-python) | Python | 386 | 2,473 | 0.3s |
| [flask](https://github.com/pallets/flask) | Python | 63 | 872 | <0.1s |
| [cobra](https://github.com/spf13/cobra) | Go | 15 | 249 | <0.1s |
| [gin](https://github.com/gin-gonic/gin) | Go | 41 | 574 | <0.1s |
| Unity game (private) | C# | 129 | 1,018 | 0.1s |

## Line Numbers for Targeted Reads

Every record includes `line_start` and `line_end` (1-indexed). Multi-line declarations span the full signature:

```
[METHOD] com.google.common.base.Converter.from
  Signature: static Converter<A, B> from(Function<...> forward, Function<...> backward)
  File: Converter.java:504-506          ← multi-line signature

[METHOD] server.AlbumController.createAlbum
  Signature: createAlbum(@Auth() auth: AuthDto, @Body() dto: CreateAlbumDto)
  File: album.controller.ts:46          ← single-line
```

This lets AI agents do **targeted reads** instead of reading full files:

```python
# Instead of reading the entire 600-line file:
Read("Converter.java")                     # 600 lines, ~12k tokens

# Read just the method + context:
Read("Converter.java", offset=504, limit=10)  # 10 lines, ~200 tokens
```

## Benchmarks

Measured across 5 real-world projects in 5 languages, each using a 10-step cross-cutting research workflow.

![Total Tokens — Cross-Language Comparison](https://raw.githubusercontent.com/Codeturion/codesurface/master/docs/images/01-total-tokens.png)

| Language | Project | Files | Records | MCP | Skilled | Naive | MCP vs Skilled |
|----------|---------|------:|--------:|----:|--------:|------:|---------------:|
| C# | Unity game | 129 | 1,034 | **1,021** | 4,453 | 11,825 | 77% fewer |
| TypeScript | immich | 694 | 8,344 | **1,451** | 4,500 | 14,550 | 68% fewer |
| Java | guava | 891 | 8,377 | **1,851** | 4,200 | 26,700 | 56% fewer |
| Go | gin | 38 | 534 | **1,791** | 2,770 | 15,300 | 35% fewer |
| Python | codesurface | 9 | 40 | **753** | 2,000 | 10,400 | 62% fewer |

![Hallucination Risk](https://raw.githubusercontent.com/Codeturion/codesurface/master/docs/images/04-hallucination.png)

Even with follow-up reads for implementation detail, the hybrid MCP + targeted Read approach uses **44% fewer tokens** than a skilled Grep+Read agent and **87% fewer** than a naive agent:

![Hybrid Workflow](https://raw.githubusercontent.com/Codeturion/codesurface/master/docs/images/03-hybrid.png)

### Per-question breakdown

![Per Question](https://raw.githubusercontent.com/Codeturion/codesurface/master/docs/images/02-per-step.png)

See [workflow-benchmark.md](workflow-benchmark.md) for the full step-by-step analysis across all languages.

## Multiple Projects

Each `--project` flag indexes one directory. To index multiple codebases, run separate instances with different server names:

```json
{
  "mcpServers": {
    "codesurface-backend": {
      "command": "uvx",
      "args": ["codesurface", "--project", "/path/to/backend/src"]
    },
    "codesurface-frontend": {
      "command": "uvx",
      "args": ["codesurface", "--project", "/path/to/frontend/src"]
    }
  }
}
```

Each instance gets its own in-memory index and tools. The AI agent sees both and can query across projects.

## Setup Details

<details>
<summary>Alternative installation methods</summary>

**Using pip install:**
```bash
pip install codesurface
```
```json
{
  "mcpServers": {
    "codesurface": {
      "command": "codesurface",
      "args": ["--project", "/path/to/your/src"]
    }
  }
}
```

</details>

<details>
<summary>Project structure</summary>

```
codesurface/
├── src/codesurface/
│   ├── server.py           # MCP server — 5 tools
│   ├── db.py               # SQLite + FTS5 database layer
│   └── parsers/
│       ├── base.py         # BaseParser ABC
│       ├── csharp.py       # C# parser
│       ├── go.py           # Go parser
│       ├── java.py         # Java parser
│       ├── python_parser.py # Python parser
│       └── typescript.py   # TypeScript/TSX parser
├── pyproject.toml
└── README.md
```

</details>

<details>
<summary>Troubleshooting</summary>

**"No codebase indexed"**
- Ensure `--project` points to a directory containing supported source files (`.cs`, `.go`, `.java`, `.py`, `.ts`, `.tsx`)
- The server indexes at startup — check stderr for the "Indexed N records" message

**Server won't start**
- Check Python version: `python --version` (needs 3.10+)
- Check `mcp[cli]` is installed: `pip install mcp[cli]`

**Stale results after editing source files**
- The index auto-refreshes on query misses — if you add a new class and query it, the server reindexes and retries automatically
- You can also call `reindex()` manually to force an incremental update

</details>

---

## Contact

fuatcankoseoglu@gmail.com

## License

[PolyForm Noncommercial 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0/)

Free to use, fork, modify, and share for any personal or non-commercial purpose.
Commercial use requires permission.
