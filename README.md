# ascend-gha-runners/artifact

GitHub Actions for uploading and downloading build artifacts via Huawei Cloud OBS.
Drop-in replacement for `actions/upload-artifact` + `actions/download-artifact`.

---

## Upload

Upload a file or directory as a named artifact. The name is used to retrieve it later in the same workflow run.

```yaml
- name: Upload artifact
  uses: ascend-gha-runners/artifact/upload@v0.2
  with:
    access_key: ${{ secrets.HW_OBS_AK }}
    secret_key: ${{ secrets.HW_OBS_SK }}
    name: my-artifact          # artifact name, used for download
    path: ./dist/              # file or directory to upload
```

### Upload inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `access_key` | Yes | — | Huawei Cloud AK |
| `secret_key` | Yes | — | Huawei Cloud SK |
| `name` | Yes | — | Artifact name |
| `path` | Yes | — | File or directory to upload |
| `if-no-files-found` | No | `warn` | What to do if `path` does not exist: `warn` / `error` / `ignore` |
| `retention-days` | No | `90` | Recorded in metadata only, not enforced by OBS |
| `bucket` | No | `ascend-ci-cache-hk` | OBS bucket |
| `endpoint` | No | `obs.ap-southeast-1.myhuaweicloud.com` | OBS endpoint |

### Upload outputs

| Output | Description |
|--------|-------------|
| `artifact-id` | Artifact identifier: `{owner}/{repo}/{run_id}/{name}` |
| `artifact-url` | OBS URL of the uploaded artifact |

---

## Download

### By exact name

Download a single artifact by its exact name.

```yaml
- name: Download artifact
  uses: ascend-gha-runners/artifact/download@v0.2
  with:
    access_key: ${{ secrets.HW_OBS_AK }}
    secret_key: ${{ secrets.HW_OBS_SK }}
    name: my-artifact
    path: ./output/
```

Result:
```
output/
└── <uploaded files>
```

### By glob pattern

Download multiple artifacts whose names match a pattern.

```yaml
- name: Download matching artifacts
  uses: ascend-gha-runners/artifact/download@v0.2
  with:
    access_key: ${{ secrets.HW_OBS_AK }}
    secret_key: ${{ secrets.HW_OBS_SK }}
    pattern: timing-data-*     # matches timing-data-singlecard, timing-data-2card, etc.
    path: ./output/
```

Result (default `merge-multiple: false`):
```
output/
├── timing-data-singlecard/
│   └── timing.json
└── timing-data-2card/
    └── timing.json
```

### merge-multiple

When downloading multiple artifacts, `merge-multiple: true` places all files directly into `path/` instead of separate subdirectories. Only useful when the artifacts do not contain files with the same name.

```yaml
- uses: ascend-gha-runners/artifact/download@v0.2
  with:
    ...
    pattern: timing-data-*
    path: ./output/
    merge-multiple: true
```

Result:
```
output/
├── singlecard_timing.json
└── twocard_timing.json
```

### Download inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `access_key` | Yes | — | Huawei Cloud AK |
| `secret_key` | Yes | — | Huawei Cloud SK |
| `name` | No* | — | Exact artifact name |
| `pattern` | No* | — | Glob pattern, e.g. `timing-data-*` |
| `path` | No | `.` | Local destination directory |
| `merge-multiple` | No | `false` | Merge all files into `path/` instead of `path/{name}/` subdirectories |
| `run-id` | No | current run | Download artifacts from a specific run ID |
| `repository` | No | current repo | Download artifacts from a specific repository (`owner/repo`) |
| `bucket` | No | `ascend-ci-cache-hk` | OBS bucket |
| `endpoint` | No | `obs.ap-southeast-1.myhuaweicloud.com` | OBS endpoint |

\* Exactly one of `name` or `pattern` is required.

### Download outputs

| Output | Description |
|--------|-------------|
| `download-path` | Absolute path to the directory where artifacts were downloaded |
| `matched-artifacts` | Space-separated list of artifact names that were downloaded |

---

## Cross-job example

A common pattern: multiple jobs produce artifacts, a final job collects them all.

```yaml
jobs:
  test-singlecard:
    runs-on: linux-aarch64-a3-1
    steps:
      - uses: actions/checkout@v4

      - name: Run benchmark
        run: python benchmark.py --cards 1 --output timing.json

      - name: Upload timing data
        uses: ascend-gha-runners/artifact/upload@v0.2
        with:
          access_key: ${{ secrets.HW_OBS_AK }}
          secret_key: ${{ secrets.HW_OBS_SK }}
          name: timing-data-singlecard
          path: timing.json

  test-2card:
    runs-on: linux-aarch64-a3-2
    steps:
      - uses: actions/checkout@v4

      - name: Run benchmark
        run: python benchmark.py --cards 2 --output timing.json

      - name: Upload timing data
        uses: ascend-gha-runners/artifact/upload@v0.2
        with:
          access_key: ${{ secrets.HW_OBS_AK }}
          secret_key: ${{ secrets.HW_OBS_SK }}
          name: timing-data-2card
          path: timing.json

  update-estimates:
    needs: [test-singlecard, test-2card]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Download all timing artifacts
        uses: ascend-gha-runners/artifact/download@v0.2
        with:
          access_key: ${{ secrets.HW_OBS_AK }}
          secret_key: ${{ secrets.HW_OBS_SK }}
          pattern: timing-data-*
          path: ./timing/
          # merge-multiple: false (default) — each artifact in its own subdirectory

      - name: Update estimated times
        run: |
          python update_estimated_times.py \
            --singlecard timing/timing-data-singlecard/timing.json \
            --twocard    timing/timing-data-2card/timing.json
```

---

## OBS storage layout

Artifacts are stored under a deterministic path so any job in the same run can locate them:

```
obs://ascend-ci-cache-hk/
└── artifacts/
    └── {owner}/{repo}/
        └── {run_id}/
            ├── _meta/
            │   ├── timing-data-singlecard.json   ← metadata index
            │   └── timing-data-2card.json
            ├── timing-data-singlecard/
            │   └── timing.json                   ← actual files
            └── timing-data-2card/
                └── timing.json
```

The `_meta/` directory is what `pattern` matching scans to discover available artifacts. Each metadata file records the artifact name, size, file count, creation time, and OBS path.
