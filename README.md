# python-action

[![test](https://github.com/rcmdnk/python-action-test/actions/workflows/test.yml/badge.svg)](https://github.com/rcmdnk/python-action-test/actions/workflows/test.yml)

GitHub Action for Python (Poetry, pytest with coverage and  linters with pre-commit).

## Examples

    steps:
    - uses: rcmdnk/python-action@v1

## Inputs

Name | Description | Default | Required
-|:-|-|-
checkout| Set '1' to run [checkout](https://github.com/marketplace/actions/checkout). | '1' | No
set-python| Set '1' to run [setup-pytyon](https://github.com/marketplace/actions/setup-python). | '1' | No
python-version| `python-version` for setup-python. | '3.10' | No
poetry| Set '1' to set up by poetry. | '1' | No
cache| Set '1' to use cache with [cache](https://github.com/marketplace/actions/cache) | '~/.cache/pypoetry' | No
cache-path| Cache path. | '~/.cache/pypoetry' | No
cache-hash-file| File name to be used for the cache hash key. | '~/.cache/pypoetry' | No
pytest| Set '1' to run pytest | '1' | No
pytest-tests-path| Path to the directory of the test files.| 'tests/' | No
pytest-ignore| Comma separated test files which are excluded from the pytest. |'' | No
pytest-cov-path| Path to check coverage.| 'src' | No
coverage | Set '1' to check coverage for pytest. | '1' | No
coverage-push | Set '1' to push the coverage result to `coverage` branch. | '0' | No
github_token | Token to push `coverage` branch. Set `secrets.GITHUB_TOKEN` if you set '1' for coverage.| "${{ github.token }}" | No
pre-commit | Set '1' to run pre-commit. | '1' | No

If you want to enable `coverage`,
add `jobs.<job_id>.permissions.contents: write` in your workflow file
or
go *`*Settings** -> **Code automation, Actions, General**,  
and set **Read and write permissions** and check **Allow GitHub Actions to create and approve pull requests** in **Workflow permissions**.

Ref:

* [Automatic token authentication - GitHub Docs](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#permissions-for-the-github_token)


