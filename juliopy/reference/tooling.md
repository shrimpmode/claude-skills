# pyproject.toml tooling config

Set `target-version`/`python_version` to the Python version resolved in SKILL.md step 2.

## [tool.ruff]

```toml
[tool.ruff]
target-version = "py<python-short>"  # e.g. "py313"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "SIM", "C4"]
ignore = []

[tool.ruff.format]
quote-style = "double"
```

`select` groups: `E`/`F` (pycodestyle/pyflakes core), `I` (isort — replaces a separate import sorter), `B` (bugbear — catches real bugs, not just style), `UP` (pyupgrade — flags outdated syntax for the target Python), `C4` (comprehension cleanups).

## [tool.mypy]

```toml
[tool.mypy]
python_version = "<python-short>"
strict = true
warn_unused_ignores = true
```

For Django, add `django-stubs` as a dev dependency and:

```toml
[tool.mypy]
plugins = ["mypy_django_plugin.main"]

[tool.django-stubs]
django_settings_module = "config.settings"
```

If `strict = true` produces too much friction on a fast-moving prototype, relax to explicit flags (`disallow_untyped_defs`, `check_untyped_defs`, `no_implicit_optional`) instead of turning strictness off wholesale — say so and note the tradeoff rather than silently downgrading.

`strict = true` means every view/handler function and every test function needs a full signature (e.g. `def health(request: HttpRequest) -> JsonResponse:`, `def test_health(client: Client) -> None:`) — budget for that, it's not optional cleanup.

### pydantic-settings + mypy strict: expected false positive

A required field like `database_url: str` with no default makes mypy strict flag `Settings()` with `Missing named argument "database_url" for "Settings"` (`call-arg`) — the values are actually supplied at runtime from the environment/`.env`, not as constructor arguments, so this is a known, harmless mismatch between pydantic-settings' dynamic loading and mypy's static view of `__init__`. Silence it at the one call site rather than loosening `strict`:

```python
settings = Settings()  # type: ignore[call-arg]  # values come from .env / environment
```

## [tool.pytest.ini_options]

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
```

FastAPI only, add:

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

Django only, add `DJANGO_SETTINGS_MODULE` and add `pytest-django` as a dev dependency (`uv add --dev pytest-django`) — bare pytest has no `db` marker, no `client` fixture, and no Django app registry, so Django tests fail to collect without it:

```toml
[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "config.settings"
```
