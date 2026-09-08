---
name: viverse-pls-cli
description: Upload and replace 3D model assets to VIVERSE using pls-cli. Use when the task involves uploading .zip/.glb/.obj files to VIVERSE, replacing existing assets, managing model conversion, or running pls-cli commands against stage/prod environments.
---

# pls-cli — VIVERSE Model Upload/Replace CLI

Operational guide for AI agents running pls-cli to upload or replace 3D models on VIVERSE.

## When to Activate

- User wants to upload a model file (.zip, .glb, .obj) to VIVERSE
- User wants to replace an existing asset by asset ID
- Running smoke tests or integration tests against stage API
- Debugging upload/conversion failures

---

## 0. Install pls-cli

### Check if already installed

```bash
which pls-cli || ls ~/bin/pls-cli 2>/dev/null || ls /usr/local/bin/pls-cli 2>/dev/null
pls-cli version 2>/dev/null || true
```

If a binary exists, still check the version. Do **not** keep `v1.0.0` when a newer release is available.

### Install from GitHub Releases (recommended)

Pin to a checked GitHub release. As of 2026-09, use **`v1.2.0`**. Confirm the latest tag at `https://github.com/ViveportSoftware/pls-cli/releases` before installing; never default to `v1.0.0`.

Detect OS and architecture, then download the correct binary:

```bash
OS=$(uname -s | tr '[:upper:]' '[:lower:]')   # darwin or linux
ARCH=$(uname -m)                               # x86_64 or arm64
VERSION="v1.2.0"                               # verified 2026-09; bump after checking GitHub Releases

# Normalise arch name
case "$ARCH" in
  x86_64)  ARCH="amd64" ;;
  arm64|aarch64) ARCH="arm64" ;;
esac

BINARY="pls-cli-${OS}-${ARCH}"
URL="https://github.com/ViveportSoftware/pls-cli/releases/download/${VERSION}/${BINARY}"

mkdir -p ~/bin
curl -fsSL "$URL" -o ~/bin/pls-cli
chmod +x ~/bin/pls-cli
```

> **Windows**: Download `pls-cli-windows-amd64.exe` from the releases page and add it to your PATH.

### Add ~/bin to PATH (if not already)

```bash
# Check if ~/bin is in PATH
echo $PATH | grep -q "$HOME/bin" || export PATH="$HOME/bin:$PATH"

# To persist, add to ~/.zshrc or ~/.bashrc:
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc
```

### Verify installation

```bash
pls-cli version
```

---

## 1. Before Running Commands

### Credential safety rule

NEVER pass email or password as literal values in shell commands — they appear in logs.
Always read from env vars. Canonical names:

- `PLS_CLI_TEST_EMAIL`
- `PLS_CLI_TEST_PASSWORD`
- `PLS_CLI_TEST_GROUP_UUID` (optional; otherwise auto-select or pass `--group`)

Some repos store the same account as `VIVERSE_TEST_EMAIL` / `VIVERSE_TEST_PASSWORD` (for example e2e env files). Map those before login; do not print the values:

```bash
# Prefer PLS_CLI_* when set; otherwise accept VIVERSE_TEST_* aliases.
export PLS_CLI_TEST_EMAIL="${PLS_CLI_TEST_EMAIL:-$VIVERSE_TEST_EMAIL}"
export PLS_CLI_TEST_PASSWORD="${PLS_CLI_TEST_PASSWORD:-$VIVERSE_TEST_PASSWORD}"
```

If neither pair exists, ask the user to set the variables in their terminal before proceeding.

---

## 2. Authentication

The CLI uses cookie-based auth stored in `~/.pls-cli/credentials.json`.
There is **no token env var** — you must log in first with `pls-cli login`.

### Login (saves credentials to ~/.pls-cli/credentials.json)

```bash
# Stage
pls-cli login --stage \
  --email="$PLS_CLI_TEST_EMAIL" \
  --password="$PLS_CLI_TEST_PASSWORD"

# Production
pls-cli login \
  --email="$PLS_CLI_TEST_EMAIL" \
  --password="$PLS_CLI_TEST_PASSWORD"
```

### Verify login succeeded

```bash
pls-cli status
# Outputs: email, account ID, environment (stage/prod), token expiry
```

### Environment mismatch — handled automatically

If you run `upload --stage` but logged in for prod (or vice versa), the CLI will **exit with an error before touching the API**:

```
error: credentials are for prod environment, but --stage flag was provided
Re-run: pls-cli login --stage
```

You do not need to manually check the environment — the CLI enforces it.

---

## 3. Upload

```bash
# Minimal — --group is OPTIONAL (CLI auto-selects your first group if omitted)
pls-cli upload model.zip

# With explicit group
pls-cli upload model.zip --group=<group-uuid>

# Stage environment
pls-cli upload model.zip --group=<group-uuid> --stage

# With conversion options
pls-cli upload model.glb \
  --group=<group-uuid> \
  --stage \
  --ai-enhance \
  --collider \
  --resolution=high \
  --collider-scale=5

# Multi-file (max 10 files)
pls-cli upload file1.zip file2.glb file3.obj --group=<group-uuid>

# Machine-readable output (for agent parsing — recommended)
pls-cli upload model.zip --json
```

### Upload flags reference

| Flag               | Values                          | Default       | Notes                                             |
| ------------------ | ------------------------------- | ------------- | ------------------------------------------------- |
| `--group`          | UUID                            | auto-selected | Omit to use your first group automatically        |
| `--stage`          | bool                            | false         | Use staging environment                           |
| `--ai-enhance`     | bool                            | false         | AI enhancement                                    |
| `--collider`       | bool                            | false         | Generate collision mesh                           |
| `--resolution`     | performance/balanced/high/ultra | balanced      | -                                                 |
| `--collider-scale` | 0.3/2/5/10/100                  | 2.0           | -                                                 |
| `--secure`         | bool                            | false         | Encryption                                        |
| `--json`           | bool                            | false         | Structured result is JSON, but **upload progress bars may still leak onto stdout**. Parse the trailing `{...}` object, do not `JSON.parse` the whole stream. Human messages also go to stderr. |
| `--tags`           | comma-separated names           | -             | Auto-create missing tags and assign after upload  |

---

## 4. Replace

```bash
# Replace existing asset by ID
pls-cli replace <old-asset-id> new-model.zip

# Stage
pls-cli replace <old-asset-id> new-model.glb --stage

# With collider + machine-readable output
pls-cli replace <old-asset-id> new-model.obj --collider --collider-scale=10 --json
```

Replace shares the same flags as upload except `--group` (originId is provided instead). This includes `--tags` — pass comma-separated tag names to auto-create and assign tags after conversion.

---

## 5. Tag Management

Tags are labels you can attach to assets. You can create them, list them, and assign them to assets after upload.

### tag create

```bash
# Create a tag in a group
pls-cli tag create --group=<group-uuid> "my-tag"

# Stage environment
pls-cli tag create --group=<group-uuid> --stage "my-tag"

# Machine-readable output
pls-cli tag create --group=<group-uuid> --json "my-tag"
```

### tag list

```bash
# List all tags in a group
pls-cli tag list --group=<group-uuid>

# Stage environment
pls-cli tag list --group=<group-uuid> --stage

# Machine-readable output
pls-cli tag list --group=<group-uuid> --json
```

### tag assign

```bash
# Assign one or more tags to an asset (positional args are tag UUIDs)
pls-cli tag assign --asset=<asset-uuid> <tag-uuid-1> <tag-uuid-2>

# Stage environment
pls-cli tag assign --asset=<asset-uuid> --stage <tag-uuid-1>

# Machine-readable output
pls-cli tag assign --asset=<asset-uuid> --json <tag-uuid-1> <tag-uuid-2>
```

### Tag flags reference

| Command      | Flag      | Values | Notes                                             |
| ------------ | --------- | ------ | ------------------------------------------------- |
| `tag create` | `--group` | UUID   | Required — group to create the tag in             |
| `tag create` | `--stage` | bool   | Use staging environment                           |
| `tag create` | `--json`  | bool   | Write JSON to stdout; human messages go to stderr |
| `tag list`   | `--group` | UUID   | Required — group to list tags from                |
| `tag list`   | `--stage` | bool   | Use staging environment                           |
| `tag list`   | `--json`  | bool   | Write JSON to stdout; human messages go to stderr |
| `tag assign` | `--asset` | UUID   | Required — asset to assign tags to                |
| `tag assign` | `--stage` | bool   | Use staging environment                           |
| `tag assign` | `--json`  | bool   | Write JSON to stdout; human messages go to stderr |

### --tags flag on upload and replace

Pass `--tags` to auto-create any missing tags and assign them to the asset after conversion:

```bash
# Upload with tags (auto-creates "foo" and "bar" if they don't exist)
pls-cli upload model.glb --group=<group-uuid> --tags=foo,bar

# Replace with tags
pls-cli replace <old-asset-id> new-model.glb --tags=foo,bar

# Combine with --json for machine-readable output
pls-cli upload model.glb --group=<group-uuid> --tags=foo,bar --json
```

`--tags` accepts a comma-separated list of tag **names**. The CLI:

1. Lists existing tags for the group
2. Creates any tags that don't already exist
3. Assigns all resolved tag UUIDs to the asset after conversion completes

---

## 5.5 List assets and construct the playable `.xrg` URL

Upload JSON returns `assetId` and `status` only. It does **not** return a resource URL. Agents must build the URL.

```
https://stream.viverse.com/polygon_file/<group-uuid>/<asset-id>/model.xrg
```

On stage, use `https://stream-stage.viverse.com` with the same path.

**The `<group-uuid>` prefix is the asset group UUID, not the account id from `pls-cli status`.** Existing enemy/player XRGs in a group reuse that same prefix.

### `pls-cli list`

```bash
# Human output auto-selects the first group if --group is omitted.
pls-cli list

# Machine-readable list REQUIRES --group. Without it, --json stdout can be empty
# even though the command exits 0.
pls-cli list --group=<group-uuid> --json
```

List JSON shape (v1.2.0):

```json
{
  "assets": [
    {
      "id": "asset-uuid",
      "name": "example-model",
      "status": "ready",
      "createdAt": "2026-01-01T00:00:00Z"
    }
  ]
}
```

There is still no `url` field. After upload:

1. Read `files[].assetId` from upload JSON (trailing object; see Section 6).
2. Resolve `group-uuid` from `--group`, `PLS_CLI_TEST_GROUP_UUID`, or the `auto-selected group: ... (uuid)` line on human `pls-cli list` stderr/stdout.
3. Write `https://stream.viverse.com/polygon_file/<group-uuid>/<assetId>/model.xrg` into the app manifest.
4. Verify with a **GET** Range request, not `curl -I` plus a Range header (HEAD ignores Range and returns 200):

```bash
curl -s -D - -o /tmp/xrg.bin -H 'Range: bytes=0-15' \
  "https://stream.viverse.com/polygon_file/<group-uuid>/<asset-id>/model.xrg"
# Expect HTTP 206, content-range: bytes 0-15/<len>, body starting with v.00001
```

`HEAD` without Range returning 200 is a useful existence check; `GET` 206 is the streaming check.

---

## 6. Machine-Readable Output (--json)

Always pass `--json` when the result needs to be parsed programmatically.

**Intended split:** human-readable messages on stderr; structured result on stdout.

**Actual upload behavior (v1.2.0):** S3 progress bars often print on **stdout** before the JSON object. `JSON.parse(stdout)` then fails. Extract the trailing JSON object:

```python
import json, re, sys
raw = sys.stdin.read()
match = re.search(r"\{[\s\S]*\}\s*$", raw)
data = json.loads(match.group() if match else raw)
```

Redirect stdout to a file (`> /tmp/pls-upload.json`) and inspect stderr separately so progress noise does not mix with agent logs.

### Upload JSON output

```json
{
  "files": [
    {
      "file": "model.zip",
      "assetId": "abc-123-uuid",
      "status": "ready"
    }
  ]
}
```

### Upload JSON output (with --tags)

When `--tags` is used, the upload JSON output includes a `"tags"` field:

```json
{
  "files": [
    {
      "file": "model.zip",
      "assetId": "abc-123-uuid",
      "status": "ready"
    }
  ],
  "tags": [
    { "uuid": "tag-uuid-1", "name": "foo" },
    { "uuid": "tag-uuid-2", "name": "bar" }
  ]
}
```

### tag create JSON output

```json
{ "uuid": "tag-uuid-1", "name": "my-tag" }
```

### tag list JSON output

```json
{
  "tags": [
    { "uuid": "tag-uuid-1", "name": "foo" },
    { "uuid": "tag-uuid-2", "name": "bar" }
  ]
}
```

### tag assign JSON output

```json
{ "assetId": "asset-uuid", "tagUuids": ["tag-uuid-1", "tag-uuid-2"] }
```

### Replace JSON output

```json
{
  "originId": "old-asset-uuid",
  "file": "new-model.glb",
  "assetId": "new-asset-uuid",
  "status": "ready"
}
```

### Failure case

```json
{
  "files": [
    {
      "file": "bad-model.zip",
      "assetId": "abc-123-uuid",
      "status": "failed",
      "failedType": "convert",
      "error": "Model file corrupted",
      "errorCode": "INVALID_MODEL"
    }
  ]
}
```

**Status values**: `"ready"` (success) | `"failed"` (conversion failed)

### Shell parsing example

```bash
# Check if upload succeeded. Do not JSON.parse the raw stream; progress bars may precede JSON.
pls-cli upload model.zip --json > /tmp/pls-upload.json 2>/tmp/pls-upload.err
python3 - <<'PY'
import json, re
from pathlib import Path
raw = Path("/tmp/pls-upload.json").read_text()
match = re.search(r"\{[\s\S]*\}\s*$", raw)
data = json.loads(match.group() if match else raw)
file0 = data["files"][0]
print(file0["status"], file0["assetId"])
PY
```

---

## 7. What the CLI Does Internally

Understanding this helps debug failures:

```
1. Validate file (format, size, count)
2. POST /management/asset  →  get { id, uploadUrl }
3. PUT $uploadUrl  (S3 direct upload; progress bar may print on stdout even with --json)
4. POST /management/asset/:id/convert
5. WebSocket wss://{domain}/management/user/ws  →  stream conversion progress
6. Exit 0 on "ready", exit 1 on "failed"
6.5. (optional) If --tags provided: resolve tag names → create missing tags → PUT /management/asset/:id/tags
```

Replace uses `PUT /management/asset/:originId` instead of POST at step 2.

---

## 8. Supported File Formats

| Format | Notes                         |
| ------ | ----------------------------- |
| `.zip` | Can bundle multiple resources |
| `.glb` | glTF binary                   |
| `.obj` | Wavefront OBJ                 |

Max file count: 10 per upload call.
Max file size: 500 MB per file (FREE tier limit from API).

---

## 9. Running Tests

```bash
# Unit + integration tests (race detection)
go test -race ./...

# Stage E2E tests (requires credentials)
source .env
go test -v -timeout 300s -run '^TestStage' ./cmd/pls-cli/
```

Tests auto-skip when `PLS_CLI_TEST_EMAIL` / `PLS_CLI_TEST_PASSWORD` are unset.

---

## 10. Common Failures and Fixes

| Symptom                                              | Cause                                          | Fix                                                                      |
| ---------------------------------------------------- | ---------------------------------------------- | ------------------------------------------------------------------------ |
| `401 Unauthorized`                                   | Expired token or missing cookie                | Re-run `pls-cli login`                                                   |
| `credentials are for prod, but --stage was provided` | Env mismatch at login vs upload                | Re-login with the matching `--stage` flag                                |
| Error code 11                                        | Wrong password or malformed auth ticket        | Check credentials                                                        |
| Error code 1105                                      | Binary built without correct client ID ldflags | Install the official release binary from GitHub Releases (see Section 0) |
| Error code 1108                                      | Scope not allowed                              | Don't pass extra `--scopes`                                              |
| Conversion `status: "failed"`                        | Model file corrupted or unsupported            | Check `failedType` and `errorCode` in JSON output                        |
| `JSON.parse` fails on `--json` upload stdout         | Progress bars mixed into stdout                | Parse the trailing `{...}` object, not the whole stream                  |
| `list --json` empty / invalid JSON, exit 0           | `--group` omitted                              | Pass `--group=<uuid>`; human `list` without `--group` auto-selects       |
| Built XRG 404                                        | Used account id from `status` as URL prefix    | Use the **group** UUID in `/polygon_file/<group>/<assetId>/model.xrg`    |
| Binary not found                                     | pls-cli not installed                          | Install from GitHub Releases (see Section 0)                             |
| Installed `v1.0.0` while GitHub has newer            | Skill/docs pin went stale                      | Install the current release (v1.2.0 as of 2026-09)                       |
| Dev build fails at login                             | No client ID burned in                         | Install the official release binary from GitHub Releases (see Section 0) |

---

## 11. Environments

| Env        | API base                           | WS base                          | Login flag |
| ---------- | ---------------------------------- | -------------------------------- | ---------- |
| Production | `https://stream.viverse.com`       | `wss://stream.viverse.com`       | (default)  |
| Stage      | `https://stream-stage.viverse.com` | `wss://stream-stage.viverse.com` | `--stage`  |

**Always match `--stage` between login and upload/replace.** The CLI enforces this at runtime and will exit with a clear error if they don't match.
