# Multi-repo CLOC

Run `cloc` across multiple GitHub repositories, aggregate the JSON output, and generate a Markdown summary.

## What this action does

This action:

- scans one or more repositories with `cloc`
- supports `self`, `owner/repo`, `github.com/owner/repo`, and full GitHub URLs
- can reuse the caller's checked-out workspace for the current repository
- supports per-repo `ref`, `subdir`, `excludeDir`, `clocArgs`, and `ignoredFile`
- writes raw per-repo JSON, logs, an aggregate JSON file, a manifest, and a Markdown summary
- can append the summary to the GitHub job summary and/or print it to the logs

## Requirements

- `git` must be available on the runner
- `cloc` must be available, or `install-cloc: "true"` must be enabled
- auto-install of `cloc` is only implemented for Linux runners
- if you want to scan `self` from the current workspace, you should run `actions/checkout` first

## Quick start

```yaml
name: CLOC report

on:
  workflow_dispatch:

permissions:
  contents: read

jobs:
  cloc:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout current repo
        uses: actions/checkout@v6

      - name: Run multi-repo cloc
        uses: ppipada/multi-repo-cloc@v1
        with:
          repos-json: '["self"]'
```

## Full example

```yaml
name: Multi-repo cloc report

on:
  workflow_dispatch:
    inputs:
      upload_artifact:
        description: Upload the generated .cloc-report directory as an artifact
        type: boolean
        required: false
        default: false

      fail_on_repo_error:
        description: Fail the job if one or more repos fail to clone or scan
        type: boolean
        required: false
        default: false

permissions:
  contents: read

jobs:
  cloc:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout current repo
        uses: actions/checkout@v6

      - name: Run multi-repo cloc
        id: cloc
        uses: ppipada/multi-repo-cloc@v1
        with:
          fail-on-repo-error: ${{ inputs.fail_on_repo_error }}
          output-dir: .cloc-report
          write-job-summary: "false"
          print-summary: "true"
          install-cloc: "true"
          use-current-workspace: "true"
          default-exclude-dir-json: "[]"
          default-cloc-args-json: >
            ["--vcs=git",
            "--exclude_ext=yaml,sum,mod,md5",
            "--force-lang=Mustache,tmpl",
            "--force-lang=INI,env",
            "--force-lang=INI,dev",
            "--force-lang=INI,prod",
            "--not-match-f=LICENSE$|\\.npmrc$|\\.prettierignore$|\\.gitignore$|\\.gitattributes$|\\.gitmodules$"]
          repos-json: >
            [
              {
                "repo": "self",
                "useWorkspace": true,
                "ignoredFile": "ignored.txt"
              },
              {
                "repo": "github.com/myorg/myrepo1"
              },
              {
                "repo": "github.com/myorg/myrepo2"
              },
              {
                "repo": "github.com/myorg2/myrepo3"
              },
              {
                "repo": "github.com/myorg3/myrepo4"
              }
            ]

      - name: Upload cloc artifact
        if: ${{ inputs.upload_artifact }}
        uses: actions/upload-artifact@v4
        with:
          name: multi-repo-cloc-${{ github.run_number }}
          path: ${{ steps.cloc.outputs.output_dir }}
          if-no-files-found: error

      - name: Show output paths
        run: |
          echo "output_dir=${{ steps.cloc.outputs.output_dir }}"
          echo "aggregate_json=${{ steps.cloc.outputs.aggregate_json }}"
          echo "summary_markdown=${{ steps.cloc.outputs.summary_markdown }}"
          echo "manifest_json=${{ steps.cloc.outputs.manifest_json }}"
          echo "repo_count=${{ steps.cloc.outputs.repo_count }}"
          echo "scanned_repo_count=${{ steps.cloc.outputs.scanned_repo_count }}"
          echo "failed_repo_count=${{ steps.cloc.outputs.failed_repo_count }}"
          echo "total_files=${{ steps.cloc.outputs.total_files }}"
          echo "total_blank=${{ steps.cloc.outputs.total_blank }}"
          echo "total_comment=${{ steps.cloc.outputs.total_comment }}"
          echo "total_code=${{ steps.cloc.outputs.total_code }}"
          echo "has_errors=${{ steps.cloc.outputs.has_errors }}"
```

## Repo spec formats

`repos-json` must be a JSON array.

Each item can be:

- `"self"`
- `"owner/repo"`
- `"github.com/owner/repo"`
- `"https://github.com/owner/repo"`

Or an object like:

```json
{
  "repo": "self",
  "useWorkspace": true,
  "ignoredFile": "ignored.txt"
}
```

### Repo object fields

| Field                    | Type                            | Default                    | Notes                                                                      |
| ------------------------ | ------------------------------- | -------------------------- | -------------------------------------------------------------------------- |
| `repo`                   | string                          | required                   | `self`, `owner/repo`, `github.com/owner/repo`, or full URL                 |
| `ref`                    | string                          | `default-ref`              | If set, the repo is cloned at that ref                                     |
| `token`                  | string                          | `token`                    | Token used to clone that repo                                              |
| `subdir`                 | string                          | `""`                       | Relative path within the repo to scan                                      |
| `ignoredFile`            | string                          | `""`                       | Relative path where `cloc --ignored=<path>` writes its ignored-file report |
| `useWorkspace`           | boolean                         | `use-current-workspace`    | Only applies to the current repo with no explicit ref                      |
| `excludeDir`             | array or comma-separated string | `default-exclude-dir-json` | Directory names to exclude                                                 |
| `inheritDefaultClocArgs` | boolean                         | `true`                     | If `false`, only repo-specific `clocArgs` are used                         |
| `clocArgs`               | string array                    | `[]`                       | Extra `cloc` arguments; must not include `--json` or `--out`               |

## Inputs

| Input                      | Required | Default        | Description                                                                                                         |
| -------------------------- | -------- | -------------- | ------------------------------------------------------------------------------------------------------------------- |
| `repos-json`               | yes      | none           | JSON array of repo specs                                                                                            |
| `token`                    | no       | `""`           | Default token for cloning other private repos                                                                       |
| `default-ref`              | no       | `""`           | Default ref for repos that do not specify one                                                                       |
| `default-exclude-dir-json` | no       | `[]`           | JSON array of directory names to exclude by default                                                                 |
| `default-cloc-args-json`   | no       | `[]`           | JSON array of default `cloc` args applied to each repo unless disabled per repo. Do not include `--json` or `--out` |
| `github-server-url`        | no       | `""`           | Optional GitHub server URL. Blank uses `GITHUB_SERVER_URL`                                                          |
| `output-dir`               | no       | `.cloc-report` | Output directory. Relative paths are resolved from `GITHUB_WORKSPACE`                                               |
| `use-current-workspace`    | no       | `true`         | If `true`, `self` or current `github.repository` without explicit ref can reuse the existing workspace              |
| `install-cloc`             | no       | `true`         | If `true` and `cloc` is missing, install it on Linux via `apt-get`                                                  |
| `write-job-summary`        | no       | `true`         | Append generated `summary.md` to `GITHUB_STEP_SUMMARY`                                                              |
| `print-summary`            | no       | `true`         | Print generated `summary.md` to the log                                                                             |
| `fail-on-repo-error`       | no       | `true`         | Fail the action if one or more repos fail to clone or scan                                                          |

## Outputs

| Output               | Description                                                                      |
| -------------------- | -------------------------------------------------------------------------------- |
| `output_dir`         | Output directory containing raw JSON, aggregate JSON, logs, and Markdown summary |
| `aggregate_json`     | Path to `aggregate-cloc.json`                                                    |
| `summary_markdown`   | Path to `summary.md`                                                             |
| `manifest_json`      | Path to `manifest.json`                                                          |
| `repo_count`         | Number of repos requested                                                        |
| `scanned_repo_count` | Number of repos successfully scanned                                             |
| `failed_repo_count`  | Number of repos that failed                                                      |
| `total_files`        | Aggregate `nFiles` total                                                         |
| `total_blank`        | Aggregate `blank` total                                                          |
| `total_comment`      | Aggregate `comment` total                                                        |
| `total_code`         | Aggregate `code` total                                                           |
| `has_errors`         | `true` if one or more repos failed                                               |

## Generated files

The action writes these files under `output-dir`:

- `aggregate-cloc.json`
- `summary.md`
- `manifest.json`
- `raw/*.json`
- `logs/*.stdout.log`
- `logs/*.stderr.log`

## Permissions and tokens

### Public repositories

For public repositories, `permissions: contents: read` is usually enough.

### Private repositories

If you clone other private repositories, the token you provide must have access to those repositories.

Important:

- the default `GITHUB_TOKEN` from a workflow usually only has access to the current repository
- for other private repos, use a PAT or GitHub App token with at least `contents: read` on the target repositories

Example:

```yaml
with:
  token: ${{ secrets.MULTI_REPO_READ_TOKEN }}
```

## Notes and caveats

- `install-cloc: "true"` only auto-installs `cloc` on Linux runners.
- If `use-current-workspace: "true"` is used for `self`, you should run `actions/checkout` first.
- If a repo spec includes `ref`, the action clones that repo instead of using the caller workspace.
- `subdir` and `ignoredFile` must be relative paths inside the target repo.
- `ignoredFile` is an output path for `cloc --ignored`, not a pattern file to read from.
- `default-cloc-args-json` and per-repo `clocArgs` must not include `--json` or `--out`.

## License

MIT. See [LICENSE](./LICENSE).
