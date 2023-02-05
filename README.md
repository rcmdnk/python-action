# python-action

[![test](https://github.com/rcmdnk/python-action-test/actions/workflows/test.yml/badge.svg)](https://github.com/rcmdnk/python-action-test/actions/workflows/test.yml)

GitHub Action for Python (Poetry, pytest with coverage and  linters with pre-commit).

## Inputs

Name | Description | Default | Required
-|:-|-|-
checkout| Set `1` to run [checkout](https://github.com/marketplace/actions/checkout). | `1` | No
set-python| Set `1` to run [setup-pytyon](https://github.com/marketplace/actions/setup-python). | `1` | No
python-version| `python-version` for setup-python. | `3.10` | No
pip| pip command (e.g. `pip`, `pip3`, `pip3.10`). | `pip` | No
setup-cmd| Python environment setup command (e.g. `poetry install`, `pip install .`). | `poetry install` | No
pip-packages| Space separated package names to be installed by pip. If your `setup-cmd` does not install such pytest, pytest-cov or pre-commit, set like `pytest pytest-cov pre-commit`.| ''
poetry| Set `1` to set up by poetry (`poetry` package is automatically installed by pip). | `1` | No
cache| Set `1` to use cache with [cache](https://github.com/marketplace/actions/cache) | `1` | No
cache-path| Cache path. | `~/.cache/pypoetry` | No
cache-hash-file| File name to be used for the cache hash key. The cache hash key is set as `cache-${{ runner.os }}-${{ inputs.python-version}}-${{ hashFiles(inputs.cache-hash-file) }}`| `**/poetry.lock` | No
pytest| Set `1` to run pytest | `1` | No
pytest-tests-path| Path to the directory of the test files.| `tests/` | No
pytest-ignore| Comma separated test files which are excluded from the pytest. |'' | No
pytest-cov-path| Path to check coverage.| `src` | No
coverage | Set `1` to check coverage for pytest. | `1` | No
coverage-push | Set `1` to push the coverage result to `coverage` branch. | `0` | No
coverage-push-condition | Condition to push the coverage. This will be shown in the README if it is not empty. | '' | No
github_token | Token to push `coverage` branch. Set `secrets.GITHUB_TOKEN` if you set `1` for coverage.| `${{ github.token }}` | No
pre-commit | Set `1` to run pre-commit. | `0` | No
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

### Simple example

    steps:
    - uses: rcmdnk/python-action@v1

### Example use workflow_dispatch to change variables

Prepare following two yaml files in .github/workflows:

dispatch.yml:

```yaml
---
name: dispatch

on:
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
          if [ "${{ github.ref }}" = "refs/heads/${{ inputs.main_branch }}" ] && [ "${{ matrix.os }}" = "${{ inputs.main_os }}" ] && [ "${{ matrix.python-version }}" = "${{ inputs.main_py_ver }}" ];then
            echo "IS_MAIN=1" >> $GITHUB_ENV
            is_main=1
          else
            echo "IS_MAIN=0" >> $GITHUB_ENV
            is_main=0
          fi
          if [ "${{ inputs.tmate }}" = "true" ];then
            if [ "$is_main" = 0 ];then
              echo "Tmate is enabled and this is not main, skip tests"
              exit 1
            fi
            echo "DEBUG=1" >> $GITHUB_ENV
          else
            echo "DEBUG=0" >> $GITHUB_ENV
          fi
      - uses: rcmdnk/python-action@v1
        with:
          coverage-push: "${{ env.IS_MAIN }}"
          coverage-push-condition: "branch=${{ inputs.main_branch }}, os=${{ inputs.main_os }}, python_version=${{ inputs.main_py_ver }}"
          pre-commit: "${{ env.IS_MAIN }}"
          tmate: "${{ env.DEBUG }}"
```

test.yml:

```yml
---
name: test

on:
  push:
    branches-ignore:
      - "coverage"
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    permissions:
      actions: write
    steps:
      - uses: convictional/trigger-workflow-and-wait@v1.6.5
        continue-on-error: true
        with:
          owner: ${{ github.repository_owner }}
          repo: ${{ github.event.repository.name }}
          ref: ${{ github.ref }}
          github_token: ${{ github.token }}
          workflow_file_name: dispatch.yml
      - name: Write workflow url
        run: |
          echo Main job URL: ${{ steps.dispatch.outputs.workflow_url }} >> $GITHUB_STEP_SUMMARY
      - name: Check status
        run: |
          if [ "${{ steps.dispatch.outputs.conclusion }}" != "success" ];then
            exit 1
          fi
```

* **dispatch.yml** has default values of the main condition.
* You can manually trigger dispatch.yml from `https://github.com/<owner>/<repo>/actions/workflows/dispatch.yml`
* On push/pull request, **test.yml** triggers **dispatch.yml** with the default inputs variables.
* Setup with poetry.
* inputs variables for main_branch, main_os and main_pyver
* inputs variable for tmate and run only for the main combinations.
* Set `IS_MAIN` to run `pre-commit` and `push` only for the main combination of the matrix of `os` and `python-version`.

Ref: https://github.com/rcmdnk/python-action-test/actions
