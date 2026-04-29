# Python Reference

Rules for Python work. Apply on top of [`../AGENTS.md`](../AGENTS.md).

## Style

- **PEP 8** + project formatter. Run formatter before committing.
- Type hints on all public functions and methods. `from __future__ import annotations` if pre-3.10 syntax needed.
- Prefer `pathlib.Path` over `os.path`.
- Prefer f-strings over `.format()` / `%`.
- Small functions. Pure where possible.

## Tooling

- **Package mgmt**: [`uv`](https://docs.astral.sh/uv/) — always. New projects use `uv init`. Existing projects on poetry/pip should be migrated; until then run uv against the project's `pyproject.toml`.
- **Lint**: `ruff check` (replaces flake8/isort/pyupgrade).
- **Format**: `ruff format` (preferred — single tool, same vendor as uv). `black` only if the project already uses it.
- **Type check**: `mypy` or `pyright`.
- **Test**: `pytest` (+ `pytest-cov`, `pytest-asyncio` as needed).

Project layout: `pyproject.toml` + `uv.lock` (commit the lockfile). No `requirements.txt` for new projects — generate one with `uv export` only when a deploy target requires it.

## Logging

- `logging` module, not `print`. Module-level logger: `logger = logging.getLogger(__name__)`.
- Log with `%s` placeholders: `logger.info("user %s logged in", user_id)` — keeps formatting lazy.
- No bare `except:` — catch specific exceptions.
- Emit JSON in production (`python-json-logger` or `structlog`) — see [`structured-logging.md`](structured-logging.md).

## Testing

- **pytest**. Test files `test_*.py` or `*_test.py`. Functions `test_*`.
- Use fixtures over setup/teardown. Use `tmp_path` for filesystem tests.
- Parametrize repetitive cases with `@pytest.mark.parametrize`.
- One assertion focus per test. Clear name describing scenario.
- Mock with `unittest.mock` or `pytest-mock`. Patch where used, not where defined.

## Build / Test commands

```bash
uv sync                          # install deps from uv.lock
uv add <pkg>                     # add a runtime dep
uv add --dev <pkg>               # add a dev dep
uv run pytest                    # run tests
uv run ruff check .              # lint
uv run ruff format .             # format
uv run mypy .                    # type check
uv lock --upgrade                # refresh lockfile
uv export -o requirements.txt    # only if a deploy target needs it
```

Always prefix executables with `uv run` so the project venv is used. Don't activate venvs manually — `uv run` handles it.

## Async

- Don't mix sync blocking calls inside `async def`. Use async libs (`httpx`, `aiofiles`).
- `asyncio.gather` for concurrent awaitables. Handle exceptions explicitly.

## Security

- Never hardcode secrets. Use env vars + `os.environ` or `pydantic-settings`.
- Use `secrets` module for tokens, not `random`.
- `subprocess`: pass list of args, never `shell=True` with user input.
- SQL: parameterized queries only. ORM (SQLAlchemy) preferred.

## Common pitfalls

- Mutable default arguments (`def f(x=[])`) — use `None` + check inside.
- Late-binding closures in loops — capture with default arg or `functools.partial`.
- `==` vs `is` — `is` only for `None`, `True`, `False`, sentinels.
- Forgetting `__init__.py` in packages (less critical with namespace pkgs but still common).
