# python-action

[![test](https://github.com/rcmdnk/python-action-test/actions/workflows/test.yml/badge.svg)](https://github.com/rcmdnk/python-action-test/actions/workflows/test.yml)

GitHub Action for Python (uv/Poetry, pytest with coverage, linters with prek/pre-commit).

## Inputs

Name | Description | Default | Required
-|:-|-|-
checkout| Set `1` to run [checkout](https://github.com/marketplace/actions/checkout). | `1` | No
post-checkout-command| Optional shell command(s) to run right after checkout (even if checkout is disabled). Multiple commands can be passed as a multi-line string (e.g. with YAML `\|`). The value is interpolated into a `bash` script, so pass only trusted content. | `''` | No
set-python| Set `1` to run [setup-pytyon](https://github.com/marketplace/actions/setup-python). | `1` | No
python-version| `python-version` for setup-python. | `3.12` | No
setup-type| Python environment setup type (`poetry` `uv`, or `pip`.). | `poetry` | No
setup-option| Additional options for setup type. For `poetry`, you can set additional options for `poetry install` command (e.g. `--no-dev`). | `''` | No
pytest| Set `1` to run pytest | `1` | No
pytest-tests-path| Path to the directory of the test files.| `tests/` | No
pytest-ignore| Comma separated test files which are excluded from the pytest. |'' | No
pytest-separate-benchmark| Set 1 to run benchmark tests separately (execute only once at the main test, by --benchmark-disable) and show the benchmark results in the summary (need [pytest-benchmark](https://pypi.org/project/pytest-benchmark/).|`0` | No
coverage | Set `1` to check coverage for pytest ([pytest-cov](https://pypi.org/project/pytest-cov/) will be installed if not installed). | `1` | No
pytest-cov-path| Path to check coverage.| `src` | No
pytest-opt| Additional options for pytest. | `''` | No
coverage-push | Set `1` to push the coverage result to `coverage` branch. | `0` | No
coverage-push-condition | Condition to push the coverage. This will be shown in the README if it is not empty. | '' | No
github_token | Token to push `coverage` branch.| `${{ github.token }}` | No
pre-commit | Set `1` to run pre-commit. | `0` | No
prek | Set `1` to run prek. | `0` | No
private-repo-app-client-id | GitHub App Client ID for accessing private GitHub repositories. When set together with `private-repo-app-private-key`, `private-repo-app-owner`, and `private-repo-app-repositories`, a GitHub App installation token is generated and `url.*.insteadOf` rewrites are layered on the existing git config via `GIT_CONFIG_COUNT` so that the dependency installation step (`uv sync` / `poetry install` / `pip install`) can pull from the listed private repositories. | `''` | No
private-repo-app-private-key | Private key of the GitHub App used to access private GitHub repositories. Used together with `private-repo-app-client-id`. | `''` | No
private-repo-app-owner | GitHub organization or user where the GitHub App used for private repository access is installed. Used to scope the generated token and the `insteadOf` rewrites. | `''` | No
private-repo-app-repositories | Private repositories to scope the generated token and `insteadOf` rewrites to. One repository name per line. | `''` | No
private-repo-app-debug | Set `1` to print the resolved (token-redacted) `url.*.insteadOf` git config and run `git ls-remote` against each listed repository after the rewrite is configured. | `0` | No
tmate | Set `1` to run [tmate](https://mxschmitt.github.io/action-tmate/). | `0` | No

If you want to enable `coverage`,
add `jobs.<job_id>.permissions.contents: write` in your workflow file
or
go *`*Settings** -> **Code automation, Actions, General**,
and set **Read and write permissions** and check **Allow GitHub Actions to create and approve pull requests** in **Workflow permissions**.

Ref:

* [Automatic token authentication - GitHub Docs](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#permissions-for-the-github_token)

If you set `tmate = 1`, [tmate](https://mxschmitt.github.io/action-tmate/) session will be created and you can log in to the virtual environment by ssh.

## Outputs

Name | Description
-|:-
pytest| pytest status.
pre_commit| pre-commit status.
prek| prek status.
coverage| Coverage Percantage.
color| Coverage Color.
warnings| Coverage Warnings.
errors| Coverage Errors.
failures| Coverage Failures.
skipped| Coverage Skipped.
tests| Coverage Tests.
time| Coverage Time.
notSuccessTestInfo| Not Success Test Info.

## Examples

### Simple usage

Add following step to your steps.

    - uses: rcmdnk/python-action@v1

### Example to enable parameterized manual dispatch

Full YAML example (e.g. python_test.yml):


```yaml
---
name: python test

on:
  push:
  pull_request:
  workflow_dispatch:
    inputs:
      main_branch:
        description: "Main branch for coverage/tmate."
        type: string
        required: false
        default: "main"
      main_os:
        description: "Main os for coverage/tmate."
        type: choice
        default: "ubuntu-latest"
        options:
          - "ubuntu-latest"
          - "macos-latest"
      main_py_ver:
        description: "Main python version for coverage/tmate."
        type: choice
        default: "3.10"
        options:
          - "3.10"
          - "3.9"
      tmate:
        type: boolean
        description: 'Run the build with tmate debugging enabled (https://github.com/marketplace/actions/debugging-with-tmate). This is only for main strategy and others will be stopped.'
        required: false
        default: false

env:
  main_branch: ${{ inputs.main_branch || 'main' }}
  main_os: ${{ inputs.main_os || 'ubuntu-latest' }}
  main_py_ver: ${{ inputs.main_py_ver || '3.10' }}
  tmate: ${{ inputs.tmate || 'false' }}

jobs:
  test_matrix:
    strategy:
      matrix:
        os: [ubuntu-latest,macos-latest]
        python-version: ["3.10","3.9"]
    permissions:
      contents: write
    runs-on: ${{ matrix.os }}
    steps:
      - name: Check is main
        run: |
          if [ "${{ github.ref }}" = "refs/heads/${{ env.main_branch }}" ] && [ "${{ matrix.os }}" = "${{ env.main_os }}" ] && [ "${{ matrix.python-version }}" = "${{ env.main_py_ver }}" ];then
            echo "is_main=1" >> $GITHUB_ENV
            is_main=1
          else
            echo "is_main=0" >> $GITHUB_ENV
            is_main=0
          fi
          if [ "${{ inputs.tmate }}" = "true" ];then
            if [ "$is_main" = 0 ];then
              echo "Tmate is enabled and this is not main, skip tests"
              exit 1
            fi
            echo "debug=1" >> $GITHUB_ENV
          else
            echo "debug=0" >> $GITHUB_ENV
          fi
      - uses: rcmdnk/python-action@v1
        with:
          python-version: "${{ matrix.python-version }}"
          coverage-push: "${{ env.is_main }}"
          coverage-push-condition: "branch=${{ env.main_branch }}, os=${{ env.main_os }}, python_version=${{ env.main_py_ver }}"
          prek: "${{ env.is_main }}"
          tmate: "${{ env.debug }}"
```

* You can manually trigger python test from `https://github.com/<owner>/<repo>/actions/workflows/python_test.yml`
* Setup with poetry.
* inputs variables for main_branch, main_os and main_pyver
* inputs variable for tmate and run only for the main combinations.
* Set `IS_MAIN` to run `prek` and `push` only for the main combination of the matrix of `os` and `python-version`.

Ref: https://github.com/rcmdnk/python-action-test/actions

### Installing private GitHub dependencies via a GitHub App

If your project depends on private repositories on GitHub, register a GitHub App
installed on the relevant repositories with `Contents: read` permission, then
pass its credentials to the action:

```yaml
- uses: rcmdnk/python-action@v1
  with:
    setup-type: uv
    private-repo-app-client-id: ${{ vars.MY_APP_CLIENT_ID }}
    private-repo-app-private-key: ${{ secrets.MY_APP_PRIVATE_KEY }}
    private-repo-app-owner: my-org
    private-repo-app-repositories: |
      my-private-lib
      another-private-lib
```
