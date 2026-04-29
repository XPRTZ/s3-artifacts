# S3 Artifact Actions

Drop-in replacements for [`actions/upload-artifact`](https://github.com/actions/upload-artifact) and [`actions/download-artifact`](https://github.com/actions/download-artifact) that store artifacts in your own S3-compatible bucket instead of GitHub's built-in storage.

Works with **Scaleway**, **AWS S3**, **MinIO**, **Cloudflare R2**, and **Backblaze B2**.

## Why use this?

- GitHub's artifact storage has a 90-day maximum retention. With S3 you control lifecycle rules yourself.
- On GitHub's **Free plan**, organization secrets and private reusable workflows are not available. Because this repo is public, you can reference it from any private repository without a paid plan.
- Artifacts stay in your own infrastructure — no dependency on GitHub's storage quotas.

---

## Quick start

### 1. Set up your secrets

Add these secrets to your repository via **Settings → Secrets and variables → Actions**:

| Secret | Description |
|---|---|
| `SCW_ACCESS_KEY` | Scaleway (or AWS) access key ID |
| `SCW_SECRET_KEY` | Scaleway (or AWS) secret access key |
| `SCW_BUCKET_NAME` | Name of your S3 bucket |

### 2. Add to your workflow

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: mkdir -p dist && echo "built" > dist/app.txt

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.SCW_ACCESS_KEY }}
          aws-secret-access-key: ${{ secrets.SCW_SECRET_KEY }}
          aws-region: nl-ams

      - uses: xprtz/s3-artifacts/.github/actions/s3-upload-artifact@v1
        with:
          name: my-build
          path: dist/
          bucket: ${{ secrets.SCW_BUCKET_NAME }}
          endpoint-url: https://s3.nl-ams.scw.cloud

  deploy:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.SCW_ACCESS_KEY }}
          aws-secret-access-key: ${{ secrets.SCW_SECRET_KEY }}
          aws-region: nl-ams

      - uses: xprtz/s3-artifacts/.github/actions/s3-download-artifact@v1
        with:
          name: my-build
          path: downloaded/
          bucket: ${{ secrets.SCW_BUCKET_NAME }}
          endpoint-url: https://s3.nl-ams.scw.cloud

      - run: ls downloaded/my-build/
```

---

## Reference

### `s3-upload-artifact`

| Input | Required | Default | Description |
|---|---|---|---|
| `name` | | `artifact` | Artifact name — used as S3 path segment |
| `path` | yes | | File or directory to upload |
| `bucket` | yes | | S3 bucket name |
| `endpoint-url` | | | S3 endpoint — omit for AWS, required for Scaleway/MinIO/R2 |
| `if-no-files-found` | | `warn` | `warn`, `error`, or `ignore` |
| `overwrite` | | `false` | Fail if the artifact already exists |
| `include-hidden-files` | | `false` | Include dotfiles and dot-directories |
| `archive` | | `true` | Package as `.tar.gz` before uploading |

**Outputs**

| Output | Description |
|---|---|
| `artifact-url` | Full S3 URL of the uploaded artifact |

---

### `s3-download-artifact`

| Input | Required | Default | Description |
|---|---|---|---|
| `name` | | | Artifact to download. If omitted, downloads all artifacts from this run. |
| `path` | | `$GITHUB_WORKSPACE` | Destination directory |
| `bucket` | yes | | S3 bucket name |
| `endpoint-url` | | | S3 endpoint URL |
| `run-id` | | current run | Override to download from a different run |
| `run-attempt` | | current attempt | Override to download from a specific attempt |
| `repository` | | current repo | Override to download from a different repository |

---

## S3 path layout

Every artifact is stored at:

```
s3://{bucket}/{repository}/{run_id}/{run_attempt}/{name}/
```

For example:

```
s3://my-bucket/myorg/my-repo/12345678/1/my-build/my-build.tar.gz
```

This makes it easy to browse the bucket and find exactly which artifact belongs to which run and attempt.

---

## Scaleway endpoints

| Region | Endpoint URL |
|---|---|
| Amsterdam | `https://s3.nl-ams.scw.cloud` |
| Paris | `https://s3.fr-par.scw.cloud` |
| Warsaw | `https://s3.pl-waw.scw.cloud` |
| Milan | `https://s3.it-mil.scw.cloud` |

> **Note:** Scaleway does not support OIDC federation with GitHub Actions. Use static IAM API keys scoped to your artifact bucket with `ObjectStorageFullAccess`.

---

## Cross-run downloads

You can download artifacts from a previous run by overriding `run-id`:

```yaml
- uses: xprtz/s3-artifacts/.github/actions/s3-download-artifact@v1
  with:
    name: my-build
    bucket: ${{ secrets.SCW_BUCKET_NAME }}
    endpoint-url: https://s3.nl-ams.scw.cloud
    run-id: 9876543210
    run-attempt: 1
```

---

## Retention

There is no `retention-days` input. Configure a lifecycle rule on your S3 bucket to expire objects automatically. In Scaleway this is done under **Object Storage → your bucket → Lifecycle rules**.

---

## Limitations

- Artifacts do **not** appear in the GitHub workflow run UI (the Artifacts section is only for GitHub's own storage).
- No wildcard/glob support in the `path` input — use a directory or single file.
- No `compression-level` input — gzip default is used.
- No artifact merging.

---

## Requirements

- `ubuntu-latest` runner (AWS CLI v2 is preinstalled)
- AWS CLI v2.13+ on self-hosted runners
- `aws-actions/configure-aws-credentials@v4` called before these actions
