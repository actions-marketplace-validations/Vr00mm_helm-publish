# helm-publish

A GitHub Action that packages a Helm chart and publishes it to a Helm repository hosted on GitHub Pages.

- Packages the chart using `helm package`
- Merges into the existing `index.yaml` (all previous versions are preserved)
- Supports storing `index.yaml` and `.tgz` files in separate directories
- Retries push on concurrent conflicts

## Versioning

The chart version is read from `version` in `Chart.yaml` by default. Use the `version` input to override it at publish time — typically driven by a git tag.

### Use Chart.yaml version as-is

```yaml
- uses: Vr00mm/helm-publish@v1
  with:
    chart-path: ./charts/my-app
    # version not set → uses whatever is in Chart.yaml
```

### Override version from a git tag

When you tag your repo `v1.2.3`, strip the `v` prefix and pass it to the action:

```yaml
on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    steps:
      - uses: Vr00mm/helm-publish@v1
        with:
          chart-path: ./charts/my-app
          version: ${{ github.ref_name }}  # e.g. tag v1.2.3 → chart version 1.2.3
```

Note: if your tag is `v1.2.3` and you want to strip the `v`:
```yaml
version: ${{ github.ref_name }}          # produces 1.2.3 directly (helm accepts vX.Y.Z)
# or explicitly strip the v:
# run: echo "version=${GITHUB_REF_NAME#v}" >> $GITHUB_OUTPUT
# version: ${{ steps.version.outputs.version }}
```

Every published version is preserved in `index.yaml` — users can install any past version:
```bash
helm install my-app vr00mm/my-app --version 1.2.3
```

## Usage

```yaml
- uses: Vr00mm/helm-publish@v1
  with:
    chart-path: ./charts/my-app
    token: ${{ secrets.PAGES_TOKEN }}
    pages-repo: Vr00mm/vr00mm.github.io
    pages-url: https://vr00mm.github.io
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `chart-path` | yes | | Path to the Helm chart directory (must contain `Chart.yaml`) |
| `token` | yes | | GitHub token with `contents: write` on the pages repository |
| `pages-repo` | yes | | Repository hosting the Helm repo (`owner/repo`) |
| `pages-url` | yes | | Base URL of the GitHub Pages site (e.g. `https://owner.github.io`) |
| `version` | no | from `Chart.yaml` | Chart version override |
| `pages-branch` | no | `master` | Branch of the pages repository |
| `charts-dir` | no | `.` | Subdirectory where `.tgz` files are stored. Use `.` for root. |
| `index-dir` | no | same as `charts-dir` | Subdirectory where `index.yaml` is written. Set to `.` to keep the index at root while charts live in a subdirectory. |

## Outputs

| Output | Description |
|---|---|
| `chart-package` | Filename of the packaged `.tgz` chart |

## Prerequisites

### 1. GitHub Pages repository

You need a repository with GitHub Pages enabled (e.g. `owner/owner.github.io` or any repo with a `gh-pages` branch).

### 2. PAGES_TOKEN secret

Create a **fine-grained personal access token** at https://github.com/settings/personal-access-tokens/new:

- **Repository access:** your pages repo only
- **Permissions:** `Contents` → Read and write

Add it as a secret named `PAGES_TOKEN` in the repository that calls this action.

## Examples

### Basic — index and charts at root

```yaml
- uses: Vr00mm/helm-publish@v1
  with:
    chart-path: ./charts/my-app
    token: ${{ secrets.PAGES_TOKEN }}
    pages-repo: Vr00mm/vr00mm.github.io
    pages-url: https://vr00mm.github.io
```

Users add the repo:
```bash
helm repo add vr00mm https://vr00mm.github.io
helm repo update
helm search repo vr00mm
```

### Split layout — index at root, charts in subdirectory

```yaml
- uses: Vr00mm/helm-publish@v1
  with:
    chart-path: ./charts/my-app
    token: ${{ secrets.PAGES_TOKEN }}
    pages-repo: Vr00mm/vr00mm.github.io
    pages-url: https://vr00mm.github.io
    charts-dir: charts
    index-dir: .
```

Users still add the repo the same way:
```bash
helm repo add vr00mm https://vr00mm.github.io
```

### Version override (e.g. from a git tag)

```yaml
- name: Publish Helm chart
  uses: Vr00mm/helm-publish@v1
  with:
    chart-path: ./charts/my-app
    version: ${{ github.ref_name }}   # e.g. 1.2.3 from tag v1.2.3
    token: ${{ secrets.PAGES_TOKEN }}
    pages-repo: Vr00mm/vr00mm.github.io
    pages-url: https://vr00mm.github.io
    charts-dir: charts
    index-dir: .
```

### Full release workflow example

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - name: Publish Helm chart
        uses: Vr00mm/helm-publish@v1
        with:
          chart-path: ./charts/my-app
          version: ${{ github.ref_name }}
          token: ${{ secrets.PAGES_TOKEN }}
          pages-repo: Vr00mm/vr00mm.github.io
          pages-url: https://vr00mm.github.io
          charts-dir: charts
          index-dir: .
```
