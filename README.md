# Splice Plan Export Action

A GitHub Action that renders a splice plan JSON into production-ready exports
(PDF, XLSX, CSV). It produces the same outputs as the **Export** menu in
[splice](https://splice-cad.com), but headlessly, from a plan JSON committed
to your repo.

## Quickstart

1. Commit your plan JSON somewhere in your repo (e.g. `plans/main.plan.json`).
2. Add `splice.yml` at the repo root:
   ```yaml
   design:
     plan-path: plans/main.plan.json
   exports:
     formats: [all-pages-pdf, bom-xlsx]
     output-dir: rendered
   ```
3. Add a workflow at `.github/workflows/render.yml`:
   ```yaml
   name: Render plan
   on:
     push:
       paths: ['plans/**', 'splice.yml']
     workflow_dispatch:
   jobs:
     render:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - uses: splice-cad/plan-export-action@v0
         - uses: actions/upload-artifact@v4
           with:
             name: rendered
             path: rendered/
   ```
4. Push. Your rendered files land as a workflow artifact.

## Exports produced

| Target | File | Notes |
|---|---|---|
| `all-pages-pdf` | `<plan>_all_pages.pdf` | Layout + schematic for every page |
| `bom-csv` | `<plan>_bom.csv` | Bill of materials, CSV |
| `bom-xlsx` | `<plan>_bom.xlsx` | Bill of materials, Excel |
| `bom-pdf` | `<plan>_bom.pdf` | Bill of materials, PDF |
| `connection-table-xlsx` | `<plan>_connection_table.xlsx` | Engineering connection table |
| `icd-pdf` | `<plan>_icd.pdf` | Interface Control Document |

## Inputs

Inputs on the action override the corresponding fields in `splice.yml`. Most
users only need `splice.yml`.

| Input | Description |
|---|---|
| `plan-path` | Path to plan JSON (overrides `design.plan-path`) |
| `output-dir` | Output directory (overrides `exports.output-dir`) |
| `only` | Comma-separated export targets (overrides `exports.formats`) |

## Outputs

| Output | Description |
|---|---|
| `files` | Newline-separated list of rendered file paths |

## `splice.yml` reference

```yaml
design:
  # Required. Path to the plan JSON relative to the repo root.
  plan-path: plans/main.plan.json

exports:
  # Optional. If omitted, renders all targets.
  formats:
    - all-pages-pdf
    - bom-xlsx
    - bom-csv
    - bom-pdf
    - connection-table-xlsx
    - icd-pdf
  # Optional. Defaults to "rendered".
  output-dir: rendered
```

## Recipes

### Commit rendered outputs back to the repo

Keeps a git history of how your design has evolved visually. Requires
`permissions: contents: write`.

```yaml
permissions:
  contents: write

jobs:
  render:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: splice-cad/plan-export-action@v0
      - name: Commit rendered outputs
        run: |
          git config user.name  "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add rendered/
          if git diff --staged --quiet; then
            echo "No changes to rendered/"
          else
            git commit -m "chore: update rendered outputs [skip ci]"
            git push
          fi
```

The `[skip ci]` in the commit message prevents the commit-back from re-firing
the workflow infinitely.

### Attach rendered outputs to a GitHub Release

```yaml
jobs:
  render:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: splice-cad/plan-export-action@v0
      - uses: softprops/action-gh-release@v2
        if: startsWith(github.ref, 'refs/tags/')
        with:
          files: rendered/*
```

### Render multiple plans in one workflow

Point the action at each plan in a matrix.

```yaml
jobs:
  render:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        plan: [main-harness, sub-harness-a, sub-harness-b]
    steps:
      - uses: actions/checkout@v4
      - uses: splice-cad/plan-export-action@v0
        with:
          plan-path: plans/${{ matrix.plan }}.plan.json
          output-dir: rendered/${{ matrix.plan }}
      - uses: actions/upload-artifact@v4
        with:
          name: rendered-${{ matrix.plan }}
          path: rendered/${{ matrix.plan }}
```

## Versioning

| Tag | Meaning |
|---|---|
| `@v0` | Floats to latest `v0.x.y`. Bugfix + minor updates. Pre-1.0; API may drift on minor bumps. |
| `@v0.1` | Floats to latest `v0.1.x`. Bugfix updates only. |
| `@v0.1.0` | Exact pin. Reproducible across years. |

We recommend `@v0` for active development and `@v0.1.0` (or later exact pin)
for production workflows that need reproducibility.

## Troubleshooting

**`plan-path not provided`** — no `splice.yml` at repo root and no `plan-path`
input. Add one.

**`manifest unknown`** — the referenced image tag doesn't exist. Check the
action version you're using; the image is published under
`ghcr.io/splice-cad/plan-export-action`.

**`if-no-files-found: error` triggers on the upload step** — the action ran
but produced no files. Check the render step logs; most often caused by a
plan JSON with no pages.

## License

MIT. See [LICENSE](LICENSE).
