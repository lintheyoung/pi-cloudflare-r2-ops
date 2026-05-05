---
name: cloudflare-r2
description: Manage and inspect Cloudflare R2 buckets and objects via the REST API for the PestGG account.
---

# Cloudflare R2 Operations

When asked to manage or inspect Cloudflare R2 buckets or objects, follow this workflow.

## Prerequisites

Ensure these environment variables are set in the user's shell profile (e.g. `~/.zshrc`):

```bash
export CLOUDFLARE_API_TOKEN="cfut_..."
export CLOUDFLARE_ACCOUNT_ID="bf7302689d0dd0a365e5199aee2d3192"
```

The API token must have the following permissions:
- `Account:Read` – read account info
- `Zone:Read` – list zones (shared with DNS operations)
- `DNS:Read` / `DNS:Edit` – DNS operations (shared with DNS package)
- `R2 Bucket:Read` – list/read buckets
- `R2 Bucket:Edit` – create/delete buckets (if mutations are needed)
- `R2 Object:Read` – list/read objects
- `R2 Object:Edit` – upload/delete objects (if mutations needed)

**Note**: The `pi-r2` token used by PestGG team includes both DNS and R2 permissions. If you see `Authentication error` for R2 endpoints, verify your token includes `Workers R2 Storage:Read` and `Workers R2 Storage:Edit` permissions.

## API Base URL

All R2 API calls use:
```
https://api.cloudflare.com/client/v4/accounts/{ACCOUNT_ID}/r2/
```

Replace `{ACCOUNT_ID}` with `$CLOUDFLARE_ACCOUNT_ID`.

## Commands

### List buckets

```bash
curl -s -X GET "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/r2/buckets" \\
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \\
  -H "Content-Type: application/json"
```

### Create a bucket

```bash
curl -s -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/r2/buckets" \\
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \\
  -H "Content-Type: application/json" \\
  -d '{"name": "my-new-bucket"}'
```

Bucket names must be globally unique across all Cloudflare R2 and follow S3 naming rules (lowercase, alphanumeric, hyphens).

### Delete a bucket

```bash
BUCKET_NAME="my-bucket"
curl -s -X DELETE "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/r2/buckets/$BUCKET_NAME" \\
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \\
  -H "Content-Type: application/json"
```

**Bucket must be empty before deletion.** If it contains objects, deletion will fail.

### List objects in a bucket

```bash
BUCKET_NAME="my-bucket"
curl -s -X GET "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/r2/buckets/$BUCKET_NAME/objects?limit=100" \\
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \\
  -H "Content-Type: application/json"
```

### Get bucket details

```bash
BUCKET_NAME="my-bucket"
curl -s -X GET "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/r2/buckets/$BUCKET_NAME" \\
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \\
  -H "Content-Type: application/json"
```

### Get S3-compatible endpoint and credentials

```bash
curl -s -X GET "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/r2/tokens" \\
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \\
  -H "Content-Type: application/json"
```

## Safety Rules

- Never expose `CLOUDFLARE_API_TOKEN` in chat output or commit messages.
- Verify `success: true` in API responses before reporting results.
- Present results in clean markdown tables when listing buckets or objects.
- Use the fixed Account ID from the environment; do not guess.
- Bucket names are globally unique. Check availability before creation.

## HARD-GATE: Confirmation Required (CRITICAL)

Before executing ANY mutating operation (create bucket, delete bucket, delete object), you MUST:

1. Identify the exact resource(s) to be modified and display them to the user.
2. State the exact action you will take (e.g., "Delete bucket 'test-bucket' and all its objects").
3. Ask the user explicitly: "Confirm to proceed? (yes/no)"
4. WAIT for the user to respond. Do NOT execute anything while waiting.
5. ONLY execute the API call if the user replies with "yes" or "y".
6. If the user does NOT reply "yes" or "y", STOP and report: "Operation cancelled by user. No changes were made."

This gate applies to ALL of the following: creating buckets, deleting buckets, deleting objects. You are NEVER allowed to skip this confirmation step.

## Troubleshooting R2 Auth Errors

If R2 endpoints return `Authentication error` (10000):
1. The token likely lacks R2-specific permissions.
2. Go to Cloudflare Dashboard > My Profile > API Tokens.
3. Create a new token with these permissions:
   - Account: Read
   - Zone: Read
   - DNS: Read / Edit
   - R2 Bucket: Read / Edit
   - R2 Object: Read / Edit
4. Update `CLOUDFLARE_API_TOKEN` in `~/.zshrc` and run `source ~/.zshrc`.
