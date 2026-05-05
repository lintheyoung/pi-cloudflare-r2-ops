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

### List custom domains for a bucket

R2 buckets can be accessed via custom domains (e.g., `files.app.pest.gg`) instead of the default `public.r2.dev` endpoint.

```bash
BUCKET_NAME="my-bucket"
curl -s -X GET "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/r2/buckets/$BUCKET_NAME/domains/custom" \\
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \\
  -H "Content-Type: application/json"
```

### Add a custom domain to a bucket

**Prerequisites:**
1. The domain must already have DNS records pointing to Cloudflare (the zone must be managed in the same account).
2. A CNAME or A record must exist for the subdomain in DNS.

```bash
BUCKET_NAME="my-bucket"
DOMAIN="files.example.com"
ZONE_ID="your-zone-id"
curl -s -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/r2/buckets/$BUCKET_NAME/domains/custom" \\
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \\
  -H "Content-Type: application/json" \\
  -d '{
    "domain": "'"$DOMAIN"'",
    "zoneId": "'"$ZONE_ID"'",
    "minTLS": "1.2",
    "enabled": true
  }'
```

**After adding the custom domain**, you must also create a DNS CNAME record pointing the subdomain to `public.r2.dev` (or let Cloudflare handle it automatically if the zone is proxied).

### Delete a custom domain from a bucket

```bash
BUCKET_NAME="my-bucket"
DOMAIN="files.example.com"
curl -s -X DELETE "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/r2/buckets/$BUCKET_NAME/domains/custom/$DOMAIN" \\
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \\
  -H "Content-Type: application/json"
```

### Upload an object to a bucket

```bash
BUCKET_NAME="my-bucket"
OBJECT_KEY="path/to/file.txt"
LOCAL_FILE="/path/to/local/file.txt"
CONTENT_TYPE="text/plain"

curl -s -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/r2/buckets/$BUCKET_NAME/objects/$OBJECT_KEY" \\
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \\
  -H "Content-Type: $CONTENT_TYPE" \\
  --data-binary @"$LOCAL_FILE" | python3 -m json.tool
```

For binary files (images, PDFs, etc.), set the correct `Content-Type`:
- Images: `image/png`, `image/jpeg`, `image/webp`
- PDFs: `application/pdf`
- JSON: `application/json`
- ZIP: `application/zip`

### Download an object from a bucket

Objects can be downloaded via their public URL (if the bucket has public access or a custom domain configured).

First, get the object's access URL, then download:

```bash
# Option 1: Using the managed public domain (pub-xxx.r2.dev)
# First check if public access is enabled:
curl -s -X GET "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/r2/buckets/$BUCKET_NAME/domains/managed" \\
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \\
  -H "Content-Type: application/json"

# Then download using the public URL:
# curl -s -o local-file.txt "https://pub-<hash>.r2.dev/path/to/object-key"

# Option 2: Using a custom domain (e.g., files.app.pest.gg)
# curl -s -o local-file.txt "https://files.app.pest.gg/path/to/object-key"
```

### Delete an object from a bucket

```bash
BUCKET_NAME="my-bucket"
OBJECT_KEY="path/to/file.txt"

curl -s -X DELETE "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/r2/buckets/$BUCKET_NAME/objects/$OBJECT_KEY" \\
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \\
  -H "Content-Type: application/json" | python3 -m json.tool
```

### Get object public access URL

To construct the public URL for an object, first check the bucket's access configuration:

```bash
BUCKET_NAME="my-bucket"
OBJECT_KEY="path/to/file.txt"

# Check custom domains
curl -s -X GET "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/r2/buckets/$BUCKET_NAME/domains/custom" \\
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \\
  -H "Content-Type: application/json"

# Check managed (default) public domain
curl -s -X GET "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/r2/buckets/$BUCKET_NAME/domains/managed" \\
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \\
  -H "Content-Type: application/json"
```

**URL construction rules:**
- If a custom domain is configured (e.g., `files.app.pest.gg`): `https://files.app.pest.gg/{OBJECT_KEY}`
- If managed public access is enabled: `https://pub-<hash>.r2.dev/{OBJECT_KEY}`
- If neither is enabled, the object is private and cannot be accessed via public URL.

### Configure bucket public access

Enable or disable the default public access endpoint (`pub-xxx.r2.dev`) for a bucket:

```bash
BUCKET_NAME="my-bucket"

# Enable public access
curl -s -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/r2/buckets/$BUCKET_NAME/domains/managed" \\
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \\
  -H "Content-Type: application/json" \\
  -d '{"enabled": true}' | python3 -m json.tool

# Disable public access
curl -s -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/r2/buckets/$BUCKET_NAME/domains/managed" \\
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \\
  -H "Content-Type: application/json" \\
  -d '{"enabled": false}' | python3 -m json.tool
```

**Warning**: Enabling public access makes ALL objects in the bucket accessible via the public URL. Use custom domains with specific DNS records for more granular control.

### Configure bucket CORS

Get current CORS configuration:

```bash
BUCKET_NAME="my-bucket"
curl -s -X GET "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/r2/buckets/$BUCKET_NAME/cors" \\
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \\
  -H "Content-Type: application/json"
```

Update CORS configuration:

```bash
BUCKET_NAME="my-bucket"
curl -s -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/r2/buckets/$BUCKET_NAME/cors" \\
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \\
  -H "Content-Type: application/json" \\
  -d '{
    "rules": [
      {
        "allowed": {
          "origins": ["https://app.pest.gg", "https://app.momaotou.com"],
          "methods": ["GET", "HEAD", "PUT"],
          "headers": ["*"]
        },
        "exposeHeaders": ["ETag"],
        "maxAgeSeconds": 3600
      }
    ]
  }' | python3 -m json.tool
```

**CORS rule fields:**
- `allowed.origins`: List of allowed origins (use `["*"]` for all)
- `allowed.methods`: HTTP methods allowed (`GET`, `HEAD`, `PUT`, `POST`, `DELETE`)
- `allowed.headers`: Allowed request headers
- `exposeHeaders`: Headers exposed to the client
- `maxAgeSeconds`: How long browsers cache the CORS preflight response

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

This gate applies to ALL of the following: creating buckets, deleting buckets, uploading objects, deleting objects, adding custom domains, deleting custom domains, enabling/disabling public access, modifying CORS configuration. You are NEVER allowed to skip this confirmation step.

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
