# 架构文档与使用指南

## 目录

- [架构设计](#架构设计)
  - [项目结构](#项目结构)
  - [OBS 存储布局](#obs-存储布局)
  - [数据流](#数据流)
  - [元数据注册表](#元数据注册表)
- [Upload Action](#upload-action)
  - [输入参数](#upload-输入参数)
  - [输出参数](#upload-输出参数)
  - [路径语法](#路径语法)
- [Download Action](#download-action)
  - [按名称下载](#按名称下载)
  - [按模式匹配下载](#按模式匹配下载)
  - [merge-multiple 模式](#merge-multiple-模式)
  - [跨 Run / 跨仓库下载](#跨-run--跨仓库下载)
  - [输入参数](#download-输入参数)
  - [输出参数](#download-输出参数)
- [使用示例](#使用示例)
  - [单文件上传与下载](#单文件上传与下载)
  - [目录上传（带排除规则）](#目录上传带排除规则)
  - [跨 Job 传递 Artifact](#跨-job-传递-artifact)
  - [跨 Run 下载](#跨-run-下载)
- [Legacy Action (v0.1)](#legacy-action-v01)
- [FAQ](#faq)

---

## 架构设计

### 项目结构

```
ascend-gha-runners/artifact/
├── action.yml              # Legacy v0.1（仅支持单文件上传）
├── upload/
│   └── action.yml          # v0.2 Upload（支持 glob、排除、多行路径）
├── download/
│   └── action.yml          # v0.2 Download（支持精确名称、模式匹配、合并）
└── README.md
```

所有 Action 均为 **composite action**，纯 shell 脚本实现，无 Node.js 或 Docker 依赖。运行时仅需 `bash`、`curl` 和 `obsutil`（首次运行自动安装）。

### OBS 存储布局

Artifact 按确定性路径存储，同一 Run 中的任何 Job 都可以通过名称直接定位：

```
obs://{bucket}/
└── artifacts/
    └── {owner}/{repo}/
        └── {run_id}/
            ├── _meta/                              ← 元数据注册表
            │   ├── my-artifact.json
            │   ├── timing-data-singlecard.json
            │   └── timing-data-2card.json
            ├── my-artifact/                        ← Artifact 文件
            │   └── dist/
            │       ├── app.js
            │       └── app.css
            ├── timing-data-singlecard/
            │   └── timing.json
            └── timing-data-2card/
                └── timing.json
```

**核心设计决策：**

| 方面 | 设计 | 原因 |
|------|------|------|
| 路径方案 | `artifacts/{repo}/{run_id}/{name}/` | 确定性路径 — 同一 Run 中的任何 Job 无需协调即可定位 Artifact |
| 元数据 | 同一 `run_id` 下的 `_meta/{name}.json` | 支持模式发现：Download Action 扫描 `_meta/` 目录查找匹配的 Artifact |
| 文件结构 | 保留相对路径 | 上传 `testdata/sub/a.json` → 存储为 `{name}/testdata/sub/a.json` → 按 name 下载后为 `testdata/sub/a.json`（`{name}/` 前缀对用户透明） |

### 数据流

#### Upload 流程

```
  用户输入 (path)                   暂存目录                     OBS
  ──────────────                   ────────                     ───
                                   /tmp/tmp.XXXXXX/
  testdata/                   ──►    testdata/
    singlecard.json                    singlecard.json
    twocard.json                       twocard.json      ──►  obs://.../artifacts/{repo}/{run_id}/{name}/
    excluded.log             (跳过)                               testdata/singlecard.json
    subdir/                            subdir/                   testdata/twocard.json
      nested.json                        nested.json             testdata/subdir/nested.json

  !testdata/*.log            (暂存阶段应用排除规则)
```

1. **解析** — 将多行 `path` 输入拆分为包含模式和 `!` 前缀的排除模式。
2. **展开** — 通过 bash globbing（`globstar` + `nullglob`）展开每个包含模式。
3. **暂存** — 匹配的文件复制到临时暂存目录，保留相对路径结构。通过 shell `case` 检查排除模式。
4. **上传** — 将暂存目录的**内容**（非目录本身）递归上传至 `obs://{bucket}/artifacts/{repo}/{run_id}/{name}/`。
5. **注册** — 写入元数据 JSON 到 `_meta/{name}.json`，记录名称、文件数、大小、时间戳和 commit SHA。

#### Download 流程

```
  OBS                                            本地
  ───                                            ────
  obs://.../artifacts/{repo}/{run_id}/
    _meta/                                       （扫描用于模式匹配）
      artifact-a.json
      artifact-b.json
    artifact-a/                             ──►  {path}/file1.txt       （按 name 下载单个）
      file1.txt
    artifact-b/                             ──►  {path}/artifact-a/     （按 pattern 下载多个）
      file2.txt                                    file1.txt
                                                 {path}/artifact-b/
                                                   file2.txt
```

1. **解析** — 若指定 `name`，直接验证元数据 key 是否存在。若指定 `pattern`，列出所有 `_meta/*.json` 文件并通过 shell glob 过滤。
2. **下载** — 对每个匹配的 Artifact，递归下载 `artifacts/{repo}/{run_id}/{name}/` 到本地。
   - 按 `name` 下载单个 Artifact：文件直接放入 `{path}/`（与官方 `actions/download-artifact` 行为一致）。
   - 按 `pattern` 下载多个 Artifact：每个 Artifact 创建 `{path}/{name}/` 子目录。
3. **合并**（可选）— 若 `merge-multiple: true`，将每个 `{path}/{name}/` 下的文件平铺到 `{path}/` 中。

### 元数据注册表

每个上传的 Artifact 会写入一个 JSON 元数据文件到 `_meta/{name}.json`：

```json
{
  "name": "timing-data-singlecard",
  "repository": "ascend-gha-runners/vllm-ascend",
  "run_id": "24182710482",
  "object_prefix": "artifacts/ascend-gha-runners/vllm-ascend/24182710482/timing-data-singlecard",
  "file_count": 1,
  "total_size": 42,
  "created_at": "2026-04-09T10:55:00Z",
  "retention_days": 90,
  "workflow": "CI Tests",
  "job": "test-singlecard",
  "sha": "abc1234"
}
```

元数据的两个用途：
- **发现** — Download Action 扫描 `_meta/` 目录以解析 `timing-data-*` 等 glob 模式。
- **审计** — 记录每个 Artifact 的来源信息（commit SHA、workflow、job、时间戳）。

> 注意：`retention_days` 仅记录在元数据中，不会自动执行清理。需在 OBS 桶上配置生命周期规则来实现自动删除。

---

## Upload Action

```yaml
- uses: ascend-gha-runners/artifact/upload@main
  with:
    access_key: ${{ secrets.HW_OBS_AK }}
    secret_key: ${{ secrets.HW_OBS_SK }}
    name: my-artifact
    path: ./dist/
```

### Upload 输入参数

| 参数 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `access_key` | 是 | — | 华为云 Access Key (AK) |
| `secret_key` | 是 | — | 华为云 Secret Key (SK) |
| `name` | 是 | — | Artifact 名称，用于下载时检索 |
| `path` | 是 | — | 文件、目录或多行 glob 模式（见[路径语法](#路径语法)） |
| `if-no-files-found` | 否 | `warn` | 无文件匹配时的行为：`warn` / `error` / `ignore` |
| `retention-days` | 否 | `90` | 记录在元数据中（需另行配置 OBS 生命周期规则） |
| `bucket` | 否 | `ascend-ci-cache-hk` | OBS 桶名 |
| `endpoint` | 否 | `obs.ap-southeast-1.myhuaweicloud.com` | OBS 端点 |

### Upload 输出参数

| 输出 | 说明 |
|------|------|
| `artifact-id` | OBS 中的对象前缀：`artifacts/{repo}/{run_id}/{name}` |
| `artifact-url` | Artifact 在 OBS 中的完整 HTTPS URL |

### 路径语法

`path` 输入支持以下格式：

**单文件：**
```yaml
path: results/timing.json
```

**目录（递归包含所有文件）：**
```yaml
path: dist/
```

**多行模式，支持 glob 和排除：**
```yaml
path: |
  dist/
  results/*.json
  !dist/**/*.map
  !dist/**/*.log
```

规则：
- 每行一个模式
- `!` 开头的行为排除模式
- 使用 bash `globstar` 展开（`**` 匹配嵌套目录）
- 排除模式匹配相对路径
- 上传后保留原始的相对目录结构

---

## Download Action

### 按名称下载

```yaml
- uses: ascend-gha-runners/artifact/download@main
  with:
    access_key: ${{ secrets.HW_OBS_AK }}
    secret_key: ${{ secrets.HW_OBS_SK }}
    name: my-artifact
    path: ./output/
```

下载结果：
```
output/
└── <上传的文件，保留原始目录结构>
```

### 按模式匹配下载

```yaml
- uses: ascend-gha-runners/artifact/download@main
  with:
    access_key: ${{ secrets.HW_OBS_AK }}
    secret_key: ${{ secrets.HW_OBS_SK }}
    pattern: timing-data-*
    path: ./output/
```

下载结果：
```
output/
├── timing-data-singlecard/
│   └── timing.json
└── timing-data-2card/
    └── timing.json
```

模式使用 shell glob 语法匹配 `_meta/` 注册表中的 Artifact 名称。

### merge-multiple 模式

下载多个 Artifact 时，`merge-multiple: true` 会将所有文件直接平铺到 `path/` 中，去掉 `{name}/` 子目录。

```yaml
- uses: ascend-gha-runners/artifact/download@main
  with:
    access_key: ${{ secrets.HW_OBS_AK }}
    secret_key: ${{ secrets.HW_OBS_SK }}
    pattern: timing-data-*
    path: ./output/
    merge-multiple: true
```

下载结果：
```
output/
├── singlecard_timing.json
└── twocard_timing.json
```

> **注意：** 如果多个 Artifact 包含相同相对路径的文件，后下载的会覆盖先下载的。仅在确保文件名唯一时使用此选项。

### 跨 Run / 跨仓库下载

从其他 Workflow Run 或其他仓库下载 Artifact：

```yaml
- uses: ascend-gha-runners/artifact/download@main
  with:
    access_key: ${{ secrets.HW_OBS_AK }}
    secret_key: ${{ secrets.HW_OBS_SK }}
    name: baseline-results
    path: ./baseline/
    run-id: '24100000000'
    repository: 'ascend-gha-runners/other-repo'
```

### Download 输入参数

| 参数 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `access_key` | 是 | — | 华为云 Access Key (AK) |
| `secret_key` | 是 | — | 华为云 Secret Key (SK) |
| `name` | 否* | — | 精确的 Artifact 名称 |
| `pattern` | 否* | — | Glob 模式匹配 Artifact 名称（如 `timing-data-*`） |
| `path` | 否 | `.` | 本地目标目录 |
| `merge-multiple` | 否 | `false` | 按 `pattern` 下载多个 Artifact 时，将所有文件平铺到 `path/` 而非 `path/{name}/` 子目录 |
| `run-id` | 否 | 当前 Run | 从指定的 Workflow Run ID 下载 |
| `repository` | 否 | 当前仓库 | 从指定仓库下载（`owner/repo` 格式） |
| `bucket` | 否 | `ascend-ci-cache-hk` | OBS 桶名 |
| `endpoint` | 否 | `obs.ap-southeast-1.myhuaweicloud.com` | OBS 端点 |

\* `name` 和 `pattern` 必须提供其中一个，且二者互斥。

### Download 输出参数

| 输出 | 说明 |
|------|------|
| `download-path` | 下载目录的绝对路径 |
| `matched-artifacts` | 空格分隔的已下载 Artifact 名称列表 |

---

## 使用示例

### 单文件上传与下载

```yaml
jobs:
  produce:
    runs-on: ubuntu-latest
    steps:
      - run: echo '{"score": 95}' > result.json

      - uses: ascend-gha-runners/artifact/upload@main
        with:
          access_key: ${{ secrets.HW_OBS_AK }}
          secret_key: ${{ secrets.HW_OBS_SK }}
          name: test-result
          path: result.json

  consume:
    needs: produce
    runs-on: ubuntu-latest
    steps:
      - uses: ascend-gha-runners/artifact/download@main
        with:
          access_key: ${{ secrets.HW_OBS_AK }}
          secret_key: ${{ secrets.HW_OBS_SK }}
          name: test-result
          path: ./dl/

      - run: cat ./dl/result.json
        # 输出: {"score": 95}
```

### 目录上传（带排除规则）

上传整个目录，但排除日志文件和 source map：

```yaml
- uses: ascend-gha-runners/artifact/upload@main
  with:
    access_key: ${{ secrets.HW_OBS_AK }}
    secret_key: ${{ secrets.HW_OBS_SK }}
    name: build-output
    path: |
      dist/
      !dist/**/*.log
      !dist/**/*.map
```

### 跨 Job 传递 Artifact

典型 CI 场景：多个测试 Job 产出数据，汇总 Job 统一收集：

```yaml
jobs:
  test-singlecard:
    runs-on: linux-aarch64-a3-1
    steps:
      - uses: actions/checkout@v4
      - run: python benchmark.py --cards 1 --output timing.json

      - uses: ascend-gha-runners/artifact/upload@main
        with:
          access_key: ${{ secrets.HW_OBS_AK }}
          secret_key: ${{ secrets.HW_OBS_SK }}
          name: timing-data-singlecard
          path: timing.json

  test-2card:
    runs-on: linux-aarch64-a3-2
    steps:
      - uses: actions/checkout@v4
      - run: python benchmark.py --cards 2 --output timing.json

      - uses: ascend-gha-runners/artifact/upload@main
        with:
          access_key: ${{ secrets.HW_OBS_AK }}
          secret_key: ${{ secrets.HW_OBS_SK }}
          name: timing-data-2card
          path: timing.json

  aggregate:
    needs: [test-singlecard, test-2card]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: ascend-gha-runners/artifact/download@main
        with:
          access_key: ${{ secrets.HW_OBS_AK }}
          secret_key: ${{ secrets.HW_OBS_SK }}
          pattern: timing-data-*
          path: ./timing/

      - run: |
          python update_estimated_times.py \
            --singlecard timing/timing-data-singlecard/timing.json \
            --twocard    timing/timing-data-2card/timing.json
```

### 跨 Run 下载

对比当前基准测试结果与历史 Run 的基线数据：

```yaml
- name: 下载历史基线数据
  uses: ascend-gha-runners/artifact/download@main
  with:
    access_key: ${{ secrets.HW_OBS_AK }}
    secret_key: ${{ secrets.HW_OBS_SK }}
    name: timing-data-singlecard
    path: ./baseline/
    run-id: '24100000000'

- name: 下载当前结果
  uses: ascend-gha-runners/artifact/download@main
  with:
    access_key: ${{ secrets.HW_OBS_AK }}
    secret_key: ${{ secrets.HW_OBS_SK }}
    name: timing-data-singlecard
    path: ./current/

- name: 对比
  run: python compare.py --baseline baseline/ --current current/
```

---

## Legacy Action (v0.1)

根目录的 `action.yml` 提供简化的 v0.1 接口，仅支持上传单个文件。使用基于时间戳的路径方案（`github-actions/{timestamp}/{filename}`），不写入元数据。

```yaml
- uses: ascend-gha-runners/artifact@main
  with:
    access_key: ${{ secrets.HW_OBS_AK }}
    secret_key: ${{ secrets.HW_OBS_SK }}
    file_path: ./report.html
    file_name: report.html        # 可选，默认为 file_path 的文件名
```

| 输出 | 说明 |
|------|------|
| `url` | 上传文件的公开 HTTPS URL |
| `object_key` | OBS 中的对象 key |

> **注意：** v0.1 不写入元数据，与 Download Action 不兼容。新 Workflow 请使用 `upload/` + `download/`（v0.2）。

---

## FAQ

**Q: 如何配置 OBS 凭证？**
在 GitHub 仓库的 Settings > Secrets and variables > Actions 中添加 `HW_OBS_AK` 和 `HW_OBS_SK`。

**Q: Artifact 会自动清理吗？**
不会。`retention-days` 仅记录在元数据中。需要在 OBS 桶上配置生命周期规则来实现自动删除。

**Q: 可以下载其他仓库的 Artifact 吗？**
可以。在 Download Action 中使用 `repository` 和 `run-id` 参数。OBS 凭证需要对目标桶有读权限。

**Q: 同一 Run 中上传同名 Artifact 会怎样？**
后者覆盖前者。Artifact 名称在单次 Workflow Run 中应保持唯一。

**Q: 支持哪些平台？**
Linux x86_64 (`amd64`) 和 Linux AArch64 (`arm64`)。`obsutil` 在运行时自动下载对应架构的版本。不支持 macOS 和 Windows。
