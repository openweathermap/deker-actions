# Reusable GitHub CI Workflows for Deker

This repo contains reusable workflows for Deker projects. Every project is managed by uv and
drives its checks through its Makefile, so the workflows only call `make`: what CI runs is what
`make check` and `make coverage` run on a developer machine.

## Workflows

| Workflow | What it does | Inputs |
|---|---|---|
| `tox.yml` | Runs `checks.yml` on every Python of a list, and once more per Python with its lowest supported NumPy when `numpy-floors` is set | `python-versions` (JSON list), `siblings`, `numpy-floors` (JSON list) |
| `checks.yml` | `uv sync --locked`, `make check`, `make coverage`; saves the junit and coverage reports; uploads the coverage of `deker` `main` to Codecov | `python-version`, `siblings`, `numpy-version` |
| `build.yml` | `uv build`: the source distribution and the wheel, saved as the artifact `artifact` | — |
| `docs_build.yml` | `make docs-lint` and the Sphinx build, the html saved as the artifact `docs-html` | `python-version`, `version`, `ref`, `siblings` |
| `docs_publish.yml` | `docs/publish.py` of the project on the artifact: the html goes to the `gh-pages` branch as `/<version>/`, the newest version is copied to the root (`latest`), `versions.json` feeds the version switcher | `version`, `ref` |

`siblings` is a JSON list of the sibling repositories that `[tool.uv.sources]` of the project
points to (`'["deker", "deker-tools"]'`). They are cloned next to the project before `uv sync`:
the branch named like the current one when it exists, the default branch otherwise. Once the
siblings are released and the sources are replaced with version pins, the input is dropped.

The coverage threshold comes from `fail_under` in `pyproject.toml` of the project.

`numpy-floors` lists the lowest NumPy of every Python as `{python, numpy}` pairs. The lock file pins
the newest NumPy of every Python, so the matrix never sees the lower bound; with the input set, one
more job per pair installs the project with that NumPy and runs the tests with it. That environment
is not the locked one: the lock holds packages that need the newest NumPy (pandas 3 on Python 3.11
and newer), so the job resolves the project anew with `uv pip install --editable . --group dev
numpy==<version>` and the other packages come out as they do for a user who has that NumPy. The
static checks are skipped in those jobs: they are the same on any NumPy, and the type stubs of an
old NumPy are not what the code is written against. The lowest NumPy differs by Python because an
old NumPy has no wheels for a new Python: `numpy>=1.26` means 1.26.4 on Python 3.10 to 3.12, 2.1.0
on 3.13 and 2.3.2 on 3.14.

The build and the publication are separate so that a release publishes its documentation only
once its distributions are built (`publish_docs` needs `build_docs` and `build`), and the build
alone can run as a check. `docs_publish.yml` needs `permissions: contents: write` on the calling
job and GitHub Pages set to serve the `gh-pages` branch. `version` is the name the build is made
for and published under: the release tag (`v2.0.0`) or a line of releases documented once
(`1.x`). `ref` builds another tag or branch than the one that triggered the workflow, to
republish the documentation of an older version.

## Usage

```yaml
jobs:
  tests:
    uses: openweathermap/deker-actions/.github/workflows/tox.yml@master
    with:
      python-versions: '["3.10", "3.11", "3.12", "3.13", "3.14"]'
      numpy-floors: '[{"python": "3.10", "numpy": "1.26.4"}, {"python": "3.11", "numpy": "1.26.4"}, {"python": "3.12", "numpy": "1.26.4"}, {"python": "3.13", "numpy": "2.1.0"}, {"python": "3.14", "numpy": "2.3.2"}]'
      siblings: '["deker", "deker-tools"]'
    secrets: inherit  # CODECOV_TOKEN, deker only
```
