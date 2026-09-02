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
