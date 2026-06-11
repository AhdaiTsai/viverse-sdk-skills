# Parsing `pls-cli --json` Output

Read this reference before a program must consume `pls-cli --json` output.

## Output boundary

Human-readable messages are intended for stderr and structured results for stdout. In `v1.2.0`, upload progress bars can still print on stdout before the JSON object. Extract the trailing JSON object instead of calling `JSON.parse(stdout)` on the complete stream.

```python
import json, re, sys
raw = sys.stdin.read()
match = re.search(r"\{[\s\S]*\}\s*$", raw)
data = json.loads(match.group() if match else raw)
```

Redirect stdout to a file and inspect stderr separately so progress noise does not mix with agent logs.

## Upload

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

With `--tags`, upload output includes resolved tags:

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

## Tag operations

```json
{ "uuid": "tag-uuid-1", "name": "my-tag" }
```

```json
{
  "tags": [
    { "uuid": "tag-uuid-1", "name": "foo" },
    { "uuid": "tag-uuid-2", "name": "bar" }
  ]
}
```

```json
{ "assetId": "asset-uuid", "tagUuids": ["tag-uuid-1", "tag-uuid-2"] }
```

## List

```json
{
  "assets": [
    {
      "id": "asset-uuid-1",
      "name": "model.glb",
      "status": "ready",
      "createdAt": "2025-01-15T10:30:00Z"
    },
    {
      "id": "asset-uuid-2",
      "name": "scene.zip",
      "status": "converting",
      "createdAt": "2025-01-15T11:00:00Z"
    }
  ]
}
```

## Delete

Success:

```json
{ "assetId": "asset-uuid", "deleted": true }
```

Failure:

```json
{ "assetId": "asset-uuid", "deleted": false }
```

## Replace

```json
{
  "originId": "old-asset-uuid",
  "file": "new-model.glb",
  "assetId": "new-asset-uuid",
  "status": "ready"
}
```

## Conversion failure

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

Status values: `ready` for success; `failed` for conversion failure.

## Shell parsing

```bash
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
