# Upload to OBS

A GitHub Action that uploads files to Huawei Cloud OBS (Object Storage Service) and returns a public download URL.

## Usage

```yaml
- name: Upload to OBS
  id: obs
  uses: ascend-gha-runners/artifact@main
  with:
    access_key: ${{ secrets.HW_OBS_AK }}
    secret_key: ${{ secrets.HW_OBS_SK }}
    file_path: ./output.zip
    file_name: output.zip  # Optional
```

### Inputs

| Name | Required | Description |
|------|----------|-------------|
| `access_key` | Yes | Huawei Cloud Access Key (AK) |
| `secret_key` | Yes | Huawei Cloud Secret Key (SK) |
| `file_path` | Yes | Path to the file to upload |
| `file_name` | No | Target file name in OBS (defaults to source file basename) |

### Outputs

| Name | Description |
|------|-------------|
| `url` | Public download URL of the uploaded file |
| `object_key` | Object key in OBS bucket |

### Example

```yaml
jobs:
  build-and-upload:
    runs-on: linux-aarch64-a3-2
    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: |
          # ... your build steps ...
          zip -r artifact.zip ./dist

      - name: Upload artifact to OBS
        id: obs
        uses: ascend-gha-runners/artifact@main
        with:
          access_key: ${{ secrets.HW_OBS_AK }}
          secret_key: ${{ secrets.HW_OBS_SK }}
          file_path: ./artifact.zip

      - name: Print download URL
        run: echo "Download URL: ${{ steps.obs.outputs.url }}"
```

## How It Works

1. Auto-detects runner architecture (amd64/arm64) and downloads the appropriate [obsutil](https://support.huaweicloud.com/utiltg-obs/obs_11_0003.html) CLI
2. Configures OBS credentials
3. Uploads the file to `github-actions/{timestamp}/{filename}`
4. Returns the public URL

## Supported Runners

- `ubuntu-latest` (x86_64)
- Self-hosted ARM64 runners (aarch64)
