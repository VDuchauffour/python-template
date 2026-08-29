# Repository Guidelines

## Project Overview

A **[Copier](https://copier.readthedocs.io) template** that scaffolds Python projects with CI/CD, pre-commit, Renovate, and a uv-based Makefile. Usage: `copier copy gh:VDuchauffour/python-template <dest>`.

This repo is **not a Python package** — no `pyproject.toml` or `src/` at the root. Never run `uv sync`, `make install`, or `pytest` here; they fail. All renderable files live under `template/` with a `.jinja` suffix (`_templates_suffix: .jinja`, `_subdirectory: template`, Jinja extension `jinja2_time.TimeExtension`). Files without `.jinja` (e.g. `template/Makefile`) copy verbatim.

## Architecture & Data Flow

1. User answers prompts in `copier.yml` (12 variables: `project_name`, `project_description`, `github_owner`, `github_repo`, `package_name`, `python_version`, `author_name`, `author_email`, `license`, `copyright_holder`, `publish_to_pypi`, `init_git_repo`).
2. Defaults cascade: `github_repo` ← `project_name`; `package_name` ← `project_name` with `-`/space → `_`; `copyright_holder` ← `author_name` (asked only for MIT/GPL-3.0/BSD-3-Clause via `when:`).
3. Copier renders every `*.jinja` under `template/` — including the **directory name** `template/src/{{ package_name }}/` — then runs `_tasks`:
   - `publish_to_pypi=false` → deletes `.github/workflows/publish-package.yml`
   - `license=='None'` → deletes `LICENSE`
   - `init_git_repo=true` → `git init -b main` + initial commit
4. Generated projects get a hatchling + hatch-vcs package: version is fully dynamic from git tags (`_version.py` generated at build time, gitignored).

Templating patterns to preserve when editing `.jinja` files:

- Variables: `{{ project_name }}` (dist name), `{{ package_name }}` (import name), `{{ python_version }}`, `{{ github_owner }}`/`{{ github_repo }}`, `{{ author_name }}`/`{{ author_email }}`.
- Conditionals: `{% if license != 'None' %}`, `{% if publish_to_pypi %}`, `{% if author_name %}` (with nested `{% if author_email %}`).
- In GitHub Actions workflows, escape `${{ }}` expressions with `{% raw %}` blocks.

## Key Directories

| Location                           | Purpose                                                                                                                                                    |
| ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `copier.yml`                       | Prompts, defaults, `_subdirectory`, `_tasks`                                                                                                               |
| `template/`                        | Everything that ships to generated projects                                                                                                                |
| `template/src/{{ package_name }}/` | Sample package: minimal `__init__.py` (guarded `from ._version import …` with `except ModuleNotFoundError: pass`), `py.typed` (PEP 561)                    |
| `template/tests/`                  | Empty test package (`__init__.py` only); authors add `tests/test_*.py`                                                                                     |
| `template/.github/workflows/`      | CI shipped to generated projects (`ci.yml.jinja`, optional `publish-package.yml.jinja`) + repo automation (release-drafter, pr-labeler, codecov, renovate) |
| `.github/workflows/` (root)        | CI for **this template repo** only: `pr-enhancement.yml`                                                                                                   |
| `template/Makefile`                | Static (untemplated) uv-based recipes                                                                                                                      |

## Development Commands

```sh
# Render the template to a temp dir — all three flags are mandatory:
uvx --with jinja2-time copier copy . /tmp/render \
	--defaults --vcs-ref=HEAD --trust
#   --with jinja2-time → copier.yml uses jinja2_time.TimeExtension
#   --vcs-ref=HEAD     → test the working tree, not the latest git tag
#   --trust            → _tasks run shell commands (rm -f, git init)

# Verify the rendered project:
cd /tmp/render && uv sync && make check-lint && make tests

# Test a variant (no license, with PyPI publish):
uvx --with jinja2-time copier copy . /tmp/render \
	--defaults --vcs-ref=HEAD --trust \
	--data license=None --data copyright_holder= --data publish_to_pypi=true
```

Rendered-project Makefile recipes (all via `uv`, no bare pip/python):

| Command           | What it does                                                                       |
| ----------------- | ---------------------------------------------------------------------------------- |
| `make install`    | `clean` + `pre-commit-install`, then `uv venv` + `uv sync` + `uv pip install -e .` |
| `make check-lint` | `uv run ruff check .` + `uv run ruff format --check .` + `uv run ty check .`       |
| `make lint`       | Same trio in fixing mode (`ruff check --fix`, `ruff format`)                       |
| `make tests`      | `clean-test` + `install`, then `uv run pytest -vv -s` (coverage always on)         |
| `make clean`      | Removes build artifacts, `__pycache__`, test/coverage caches                       |

## Code Conventions & Common Patterns

- **Two-tier layout, never conflate**: root-level configs (`.pre-commit-config.yaml`, `.yamlfix.toml`, `.gitignore`, `.github/`) govern this template repo; the `template/` counterparts ship to generated projects. A change usually needs mirroring in both (note: they intentionally differ — e.g. root mdformat 0.7.22 vs template 1.0.0; template pre-commit also runs ruff, root does not).
- **Typing-first**: `py.typed` shipped; type checkers are `ty` (Astral) and pyright. `tool.ty.rules` marks `possibly-missing-attribute`/`possibly-missing-import` as errors. (`ty` is deliberately not yet a pre-commit hook — there's a `TODO add ty repo` comment in `template/.pre-commit-config.yaml.jinja`.)
- **Ruff**: line-length 99; target pinned from `{{ python_version|replace('.', '') }}`; select `E,W,F,S(bandit),UP,N,I`; Google-convention docstrings (`pydocstyle`), `docstring-code-format = true`. Bandit relaxed: `S101` (asserts) allowed in tests; `S301`/`S311` (pickle/random) globally ignored; `__init__.py` tolerates `E402/F401/F403/F811` for re-export style.
- **Versioning**: hatch-vcs only — never hardcode versions; `__init__.py` imports `_version` behind a `try/except ModuleNotFoundError`.
- **YAML**: yamlfix (explicit_start=false, whitelines=1, block sequences, line-length 100); TOML formatting via taplo; Markdown via mdformat (`--number`, gfm/tables/frontmatter).
- **PRs (enforced by root CI, `.github/workflows/pr-enhancement.yml`)**: conventional-commit titles (e.g. `feat:`, `fix:`), scope optional, **subject must not start with an uppercase letter**, single-commit validation; PRs labeled `dependencies` (Renovate) bypass title checks. `template/.github/pull_request_template.md` also asks for one thing per PR and a local pytest/pre-commit run.

## Important Files

- `copier.yml` — the template contract; every variable/`_task` lives here.
- `template/pyproject.toml.jinja` — single source for packaging **and** all tool config (ruff, ty, pytest, coverage, `[tool.yamlfix]`).
- `template/.github/workflows/ci.yml.jinja` — rendered CI: `lint` job (setup-uv, `make install` + `make check-lint`), then `tests` job (matrix on `{{ python_version }}`, `make tests`, Codecov upload with `fail_ci_if_error: true`).
- `template/.github/workflows/publish-package.yml.jinja` — release-triggered `uv build` + `uv publish`; deleted unless `publish_to_pypi=true`.
- `template/README.md.jinja` — badge table + install/usage sections with `publish_to_pypi`/`license` conditionals.
- `.github/workflows/pr-enhancement.yml` — root PR governance.
- `README.md` — documented option table and the jinja2-time dependency workaround.

## Runtime/Tooling Preferences

- **uv is the package manager** everywhere: `uv sync`, `uv run`, `uvx`. Python version comes from `{{ python_version }}` (template default 3.14) and `template/.python-version.jinja`.
- **No `uv.lock` committed** — gitignored at root and in `template/`; generated projects regenerate their own lockfile.
- Pre-commit hooks: install in rendered projects via `make pre-commit-install` (`uv run pre-commit install`).
- VS Code settings ship in `template/.vscode/`: Ruff as default formatter, format-on-save, explicit organize-imports; recommended extensions include Ruff, ty, Even Better TOML, autodocstring.

## Testing & QA

- **pytest ≥8 + pytest-cov ≥6** (dev dependency-group in `template/pyproject.toml.jinja`).
- `[tool.pytest.ini_options]`: `pythonpath = "src"` (tests import `{{ package_name }}` without installing), `addopts = "--cov=./src --cov-report=term --cov-report=xml --cov-report=html"` (coverage always on), `asyncio_mode = "auto"`, registered marker `integration` (unused by shipped code).
- Coverage: branch mode; omits `__init__`, tests, docs, examples, stubs; `fail_under = 0` (no local gate). The only enforcement is CI: Codecov patch threshold 5% (`template/.github/codecov.yml`) + `fail_ci_if_error: true` on upload.
- `tests/` ships as a regular package with only an empty `__init__.py` — no conftest.py, fixtures, or sample tests. New test modules follow pytest discovery (`test_*.py`).
- Validation workflow for template changes: render → `uv sync` → `make check-lint` → `make tests` (commands above). Never validate at the repo root.
