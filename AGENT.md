# Agent Guidelines for mimo_sys

This document provides guidelines for AI agents working on this codebase (Cursor, Claude Code, or any other). **Read this file first, before making any changes.** **Always keep these rules updated** as the project evolves.

## Project Overview

mimo_sys — https://github.com/watasabi/mimo_sys.git

## Tech Stack

| Component | Technology |
|-----------|------------|
| Environment | UV + Python 3.12 venv |
| Testing | pytest |
| Linting | Ruff |

## Project Structure

Note: The main package is located in `src/mimo_sys/`.

## Code Conventions

- Python 3.12+ features (type hints, `|` union, etc.)
- Line length: 79 characters
- Use `ruff` for linting and formatting

```toml
[tool.ruff]
line-length = 79

[tool.ruff.lint]
preview = true
select = ['I', 'F', 'E', 'W', 'PL', 'PT']
ignore = ['E402', 'F811']
```

- Docstrings for public APIs (Google style)
- **Always use type hints.**
- **Prefer simple, readable code over modularity.** This is a data science project: most code is exploratory or analytical, and read far more often than reused. A clear, linear script beats a premature abstraction — don't extract a function, class or registry entry unless it is already reused or the file has become hard to follow.

## Commands

Use `uv` for all operations to ensure environment isolation:

```bash
uv run ruff check src/ tests/           # Lint code
uv run ruff format src/ tests/          # Format code
uv run pytest tests/                    # Run tests
```

## Commit Message Convention

All commits **must** follow Conventional Commits. Validation regex:

```
^(Notes added by 'git notes add')|(Revert "Merge branch '\S+' into '\S+'")|( chore|docs|feat|fix|perf|refactor|imp|style|infra|test|breaking)(\(.*\))?!?: .*($|\n\n.*)
```

Allowed types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `infra`, `imp`, `breaking`

Format: `<type>(<optional scope>): <description>`

Examples:
- `feat: adiciona modelo de classificação`
- `fix(pipeline): corrige leitura de dados raw`
- `refactor(src): simplifica feature engineering`

## Important Rules

1. **Prefer simple, readable code over modularity** — see Code Conventions above. Use the registry pattern or base classes only where a real pattern of reuse already exists, not by default.
2. **Always add type hints** to function signatures.
3. **Always import from `mimo_sys`**, not relative paths.
4. **Always follow commit message convention** described above.
