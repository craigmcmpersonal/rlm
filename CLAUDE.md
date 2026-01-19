# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

RLM (Recursive Language Models) is a Python inference framework that enables LLMs to handle near-infinite length contexts by recursively calling themselves over input. It replaces standard `llm.completion()` calls with `rlm.completion()` calls that execute in a REPL environment.

## Commands

```bash
# Install dependencies
make install              # Base dependencies
make install-dev          # Dev dependencies (linter, formatter, tests)
make install-modal        # Modal sandbox support: uv pip install -e ".[modal]"

# Development
make lint                 # Run ruff linter: uv run ruff check .
make format               # Run ruff formatter: uv run ruff format .
make test                 # Run pytest: uv run pytest
make check                # Run lint + format + test

# Run single test
uv run pytest tests/test_file.py::test_name

# Pre-commit hooks (run all checks)
uv run pre-commit run --all-files

# Run examples
make quickstart           # Basic OpenAI example (needs OPENAI_API_KEY)
make docker-repl          # Docker environment example
make modal-repl           # Modal sandbox example
```

## Architecture

```
rlm/
├── clients/              # LM provider integrations (all inherit BaseLM)
│   ├── base_lm.py        # Abstract base class with completion/acompletion
│   ├── openai.py         # OpenAI/vLLM client
│   ├── anthropic.py      # Anthropic Claude client
│   └── ...               # Portkey, LiteLLM, Azure, Gemini
├── core/
│   ├── rlm.py            # Main RLM class - user-facing API
│   ├── lm_handler.py     # Multi-threaded TCP server for LM requests
│   ├── comms_utils.py    # Socket communication (length-prefixed JSON)
│   └── types.py          # Core dataclasses (RLMChatCompletion, REPLResult)
├── environments/         # Execution environments (inherit BaseEnv)
│   ├── base_env.py       # Abstract bases: NonIsolatedEnv, IsolatedEnv
│   ├── local_repl.py     # LocalREPL - same process execution (default)
│   ├── docker_repl.py    # DockerREPL - containerized execution
│   ├── modal_repl.py     # ModalREPL - cloud sandbox (HTTP broker pattern)
│   └── prime_repl.py     # PrimeREPL - Prime Intellect sandbox
├── logger/               # JSONL trajectory logging + Rich console output
└── utils/                # Parsing, prompts, utilities
```

### Key Patterns

**Plugin Architecture**: New clients inherit `BaseLM`, new environments inherit `NonIsolatedEnv` or `IsolatedEnv`. Register in respective `__init__.py`.

**Communication**:
- Non-isolated environments: TCP sockets with 4-byte length-prefixed JSON
- Isolated environments (Modal/Prime): HTTP broker pattern with `/enqueue`, `/pending`, `/respond` endpoints

**Execution Flow**: `RLM.completion()` spawns an environment and LM handler per call, iterates LM→code→execute until `FINAL_VAR()` or max iterations.

## Code Style

- **Ruff** enforced: line-length 100, double quotes, rules E/W/F/I/B/UP
- **Naming**: snake_case methods/variables, PascalCase classes, UPPER_CASE constants
- **Typing**: Explicit types preferred; `cast()`/`assert` for narrowing; avoid `# type: ignore`
- **No `_` prefix** for private methods unless explicitly requested

## Error Philosophy

**Fail fast, fail loud** - No defensive programming or silent fallbacks. Missing API key → immediate `ValueError`, not graceful fallback. Minimize branching; every `if`/`try` needs justification.

## Contributing Guidelines

- Avoid touching `core/` files unless necessary
- Keep PRs small and focused (one change per PR)
- Avoid new core dependencies; use optional extras for non-essential features
- Delete dead code (don't guard it)
- Update tests when changing functionality
- Run `make check` before PRs
