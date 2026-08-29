# Python Copier Template

A [Copier](https://copier.readthedocs.io) template for scaffolding Python projects with batteries included: CI/CD, pre-commit hooks, automated dependency updates, and a `Makefile` task runner.

## Usage

```sh
copier copy gh:VDuchauffour/python-template path/to/destination
```

> [!WARNING]
> This template relies on the [`jinja2-time`](https://pypi.org/project/jinja2-time/) extension (declared via `_jinja_extensions` in `copier.yml`). Copier will fail to render with an error about `jinja2_time.TimeExtension` if it is not installed. Install it beforehand, either as an ephemeral `uvx` run or a persistent install via `uv` / `pipx`:
>
> ```sh
> # Option 1 — inject into an ephemeral uvx run
> uvx --with jinja2-time copier copy gh:VDuchauffour/python-template path/to/destination
>
> # Option 2 — install globally once with uv
> uv tool install copier --with jinja2-time
>
> # Option 3 — install globally once with pipx
> pipx install copier
> pipx inject copier jinja2-time
> ```

Answer the prompts (project name, license, GitHub owner, etc.) and Copier renders a ready-to-develop project into `path/to/destination/`.

## What you get

- **Python** project (`uv` + `hatchling` + `hatch-vcs` for versioning, `src/` layout)
- **CI/CD** via GitHub Actions — ruff lint/format, ty type check, tests with coverage (pytest → Codecov), release drafter, conventional PR titles
- **Pre-commit** hooks — trailing whitespace, yamlfix, taplo, mdformat, ruff, prettier
- **Renovate** config for automated dependency updates
- **Makefile** recipes for common tasks (`make ci`, `make lint`, `make tests`, ...)
- **Ruff** for linting and formatting
- **ty** for type checking
- Optional **PyPI publish** workflow (toggle via `publish_to_pypi`)

## Template options

| Prompt                | Type   | Default             | Notes                                              |
| --------------------- | ------ | ------------------- | -------------------------------------------------- |
| `project_name`        | str    | `my-python-project` | Distribution name (pyproject.toml)                 |
| `project_description` | str    | `A Python project`  |                                                    |
| `github_owner`        | str    | `myusername`        |                                                    |
| `github_repo`         | str    | `= project_name`    |                                                    |
| `package_name`        | str    | `= project_name`    | Python import name (hyphens → underscores)         |
| `python_version`      | str    | `3.14`              | Minimum Python version (ruff, CI, pre-commit, ...) |
| `author_name`         | str    | _(empty)_           |                                                    |
| `author_email`        | str    | _(empty)_           |                                                    |
| `license`             | choice | `MIT`               | MIT, Apache-2.0, GPL-3.0, BSD-3-Clause, None       |
| `copyright_holder`    | str    | `= author_name`     | Asked only for MIT / GPL-3.0 / BSD-3-Clause        |
| `publish_to_pypi`     | bool   | `false`             | Adds a release-triggered PyPI publish workflow     |
