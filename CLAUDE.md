# CLAUDE.md

Instructions for Claude Code when working in this repository.

## What this repo is

Two composite GitHub Actions that replace `actions/upload-artifact@v7` and `actions/download-artifact@v8` with an S3-compatible backend (Scaleway, AWS, MinIO, R2, Backblaze). Pure bash + AWS CLI v2 — no Node.js, no Docker.

## Repository layout

```
.github/
  actions/
    s3-upload-artifact/action.yml   ← upload action
    s3-download-artifact/action.yml ← download action
  workflows/
    example.yml                     ← example workflow for consumers
```

## S3 path layout

```
s3://{bucket}/{github.repository}/{github.run_id}/{github.run_attempt}/{name}/
```

Never change this structure — consumers depend on it being stable and debuggable.

## Design constraints (don't break these)

1. **No credentials inside the actions.** Auth is always done by the caller via `aws-actions/configure-aws-credentials@v4` before our action runs.
2. **No `${{ inputs.x }}` inside bash blocks.** Pass inputs via `env:` and reference as `$INPUT_X` to prevent script injection.
3. **Use `$RUNNER_TEMP`** for temp files, never `/tmp` directly.
4. **Use `$GITHUB_OUTPUT`** for outputs — `::set-output` is deprecated.
5. **`set -e` at the top** of every multi-line bash block.
6. **No `retention-days` input** — retention is handled via bucket lifecycle rules.

## Code style

```yaml
- shell: bash
  env:
    INPUT_NAME: ${{ inputs.name }}
  run: |
    set -e
    echo "::notice::Uploading ${INPUT_NAME}"
    echo "result=done" >> "$GITHUB_OUTPUT"
```

Use `::error::`, `::warning::`, `::notice::` for runner UI feedback. Quote all variable expansions.

## Versioning

- Specific: `v1.0.0`, `v1.1.0`
- Floating major: `v1` always points to latest minor/patch

```bash
git tag -fa v1 -m "v1.0.1"
git push origin v1.0.1 v1 --force
```

## Testing checklist before releasing

- [ ] Single file upload + download
- [ ] Directory upload + download
- [ ] `archive: true` and `archive: false`
- [ ] `overwrite: false` fails when artifact exists
- [ ] `overwrite: true` replaces existing artifact
- [ ] `if-no-files-found: error` / `warn` / `ignore`
- [ ] Cross-job download in same workflow
- [ ] Cross-run download (different `run-id`)
- [ ] Hidden files excluded by default, included when `include-hidden-files: true`
- [ ] Works against Scaleway endpoint and plain AWS S3

## Known limitations (document, don't fix without discussion)

- No artifact UI in the GitHub workflow run summary
- No wildcard/glob in `path` (requires `find` + `--files-from`)
- No `compression-level` input (gzip default)
- No `retention-days` input (use bucket lifecycle rules)
- No artifact merging
