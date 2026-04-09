# ascend-gha-runners/artifact

GitHub Actions for uploading and downloading artifacts via Huawei Cloud OBS. Drop-in replacement for `actions/upload-artifact` + `actions/download-artifact`.

## v0.2 — Upload & Download with Metadata Registry

### Upload artifact

```yaml
- name: Upload artifact
  uses: ascend-gha-runners/artifact/upload@v0.2
  with:
    access_key: ${{ secrets.HW_OBS_AK }}
    secret_key: ${{ secrets.HW_OBS_SK }}
    name: timing-data-singlecard
    path: ./results/
```

### Download artifact (exact name)

```yaml
- name: Download artifact
  uses: ascend-gha-runners/artifact/download@v0.2
  with:
    access_key: ${{ secrets.HW_OBS_AK }}
    secret_key: ${{ secrets.HW_OBS_SK }}
    name: timing-data-singlecard
    path: ./downloaded/
```

### Download artifacts (glob pattern)

```yaml
- name: Download all timing artifacts
  uses: ascend-gha-runners/artifact/download@v0.2
  with:
    access_key: ${{ secrets.HW_OBS_AK }}
    secret_key: ${{ secrets.HW_OBS_SK }}
    pattern: timing-data-*
    path: ./all-timing/
    merge-multiple: true
```

---

## Cross-job example

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
        id: dl
        uses: ascend-gha-runners/artifact/download@v0.2
        with:
          access_key: ${{ secrets.HW_OBS_AK }}
          secret_key: ${{ secrets.HW_OBS_SK }}
          pattern: timing-data-*
          path: ./timing/
          merge-multiple: false   # creates timing/timing-data-singlecard/ and timing/timing-data-2card/

      - name: Update estimated times
        run: |
          python update_estimated_times.py \
            --singlecard timing/timing-data-singlecard/timing.json \
            --twocard timing/timing-data-2card/timing.json
```

---

## Input reference

### upload/action.yml

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `access_key` | Yes | — | Huawei Cloud AK |
| `secret_key` | Yes | — | Huawei Cloud SK |
| `name` | Yes | — | Artifact name (used for later retrieval) |
| `path` | Yes | — | File or directory to upload |
| `if-no-files-found` | No | `warn` | `warn` / `error` / `ignore` |
| `retention-days` | No | `90` | Recorded in metadata (not enforced by OBS) |
| `bucket` | No | `ascend-ci-cache-hk` | OBS bucket |
| `endpoint` | No | `obs.ap-southeast-1.myhuaweicloud.com` | OBS endpoint |

**Outputs:** `artifact-id`, `artifact-url`

### download/action.yml

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `access_key` | Yes | — | Huawei Cloud AK |
| `secret_key` | Yes | — | Huawei Cloud SK |
| `name` | No* | — | Exact artifact name. Mutually exclusive with `pattern` |
| `pattern` | No* | — | Glob pattern, e.g. `timing-data-*`. Mutually exclusive with `name` |
| `path` | No | `.` | Local destination directory |
| `merge-multiple` | No | `false` | If `true`, all files land directly in `path/` instead of `path/{name}/` |
| `run-id` | No | current run | Download from a different run |
| `repository` | No | current repo | Download from a different repo |
| `bucket` | No | `ascend-ci-cache-hk` | OBS bucket |
| `endpoint` | No | `obs.ap-southeast-1.myhuaweicloud.com` | OBS endpoint |

\* Exactly one of `name` or `pattern` must be provided.

**Outputs:** `download-path`, `matched-artifacts`

---

## OBS storage layout

```
obs://ascend-ci-cache-hk/
└── artifacts/
    └── {owner}/{repo}/
        └── {run_id}/
            ├── _meta/
            │   ├── timing-data-singlecard.json   ← metadata (name, size, sha, ...)
            │   └── timing-data-2card.json
            ├── timing-data-singlecard/
            │   └── timing.json                   ← actual files
            └── timing-data-2card/
                └── timing.json
```

---

## v0.1 compatibility

The root `action.yml` is unchanged. Existing workflows using `ascend-gha-runners/artifact@v0.1` or `@main` continue to work without modification.

```yaml
# v0.1 usage — still works
- uses: ascend-gha-runners/artifact@v0.1
  with:
    access_key: ${{ secrets.HW_OBS_AK }}
    secret_key: ${{ secrets.HW_OBS_SK }}
    file_path: ./output.zip
```
