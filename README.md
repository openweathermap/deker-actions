# Reusable GitHub CI Workflows for Deker

This repo contains reusable workflows for Deker projects. Every project is managed by uv and
drives its checks through its Makefile, so the workflows only call `make`: what CI runs is what
`make check` and `make coverage` run on a developer machine.

## Workflows

| Workflow | What it does | Inputs |
|---|---|---|
| `tox.yml` | Runs `checks.yml` on every Python of a list | `python-versions` (JSON list), `siblings` |
| `checks.yml` | `uv sync --locked`, `make check`, `make coverage`; saves the junit and coverage reports; uploads the coverage of `deker` `main` to Codecov | `python-version`, `siblings` |
| `build.yml` | `uv build`: the source distribution and the wheel, saved as the artifact `artifact` | — |
| `docs_build.yml` | `make docs-lint` and the Sphinx build, saved as the pages artifact | `python-version`, `siblings` |
| `docs_publish.yml` | Deploys the pages artifact to GitHub Pages | — |

`siblings` is a JSON list of the sibling repositories that `[tool.uv.sources]` of the project
points to (`'["deker", "deker-tools"]'`). They are cloned next to the project before `uv sync`:
the branch named like the current one when it exists, the default branch otherwise. Once the
siblings are released and the sources are replaced with version pins, the input is dropped.

The coverage threshold comes from `fail_under` in `pyproject.toml` of the project.

## Usage

```yaml
jobs:
  tests:
    uses: openweathermap/deker-actions/.github/workflows/tox.yml@master
    with:
      python-versions: '["3.10", "3.11", "3.12", "3.13", "3.14"]'
      siblings: '["deker", "deker-tools"]'
    secrets: inherit  # CODECOV_TOKEN, deker only
```
