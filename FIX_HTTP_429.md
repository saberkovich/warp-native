# Fix for wgcf HTTP 429 Registration Issue

## Problem Description

The `wgcf register` command was failing with **HTTP 429 Too Many Requests** error when attempting to register with Cloudflare's WARP API. This issue was reproducible across multiple VPS servers with different IP addresses and locations.

## Root Cause

Cloudflare's WARP registration API now enforces validation on the registration payload. The current `wgcf` binary (v2.2.30 - v2.2.32) sends empty values for:
- `install_id` (empty string)
- `fcm_token` (empty string)
- `tos` (Unix timestamp instead of ISO-8601 format)

While the API previously accepted these empty values, it now returns **HTTP 429** instead of the expected **HTTP 400** for invalid requests.

## Solution Implemented

Since this repository doesn't contain the `wgcf` source code (it downloads pre-compiled binaries), a **workaround** has been implemented in the `install.sh` script:

### New Function: `manual_warp_register()`

This function performs direct WARP registration using `curl` with properly formatted payload:

1. **Generates required tokens:**
   - `install_id`: 22-character random alphanumeric string
   - `fcm_token`: Firebase Cloud Messaging token format (`install_id:APA91b` + 134 random chars)
   - `tos`: ISO-8601 formatted timestamp

2. **Creates WireGuard keys:**
   - Generates private key using `wg genkey`
   - Derives public key from private key

3. **Sends registration request:**
   - Endpoint: `https://api.cloudflareclient.com/v0a1922/reg`
   - Headers: Mimics Android WARP client
   - Payload: Complete registration data with non-empty fields

4. **Parses response and creates `wgcf-account.toml`:**
   - Extracts: account ID, access token, peer public key, endpoint
   - Creates TOML file compatible with `wgcf generate` command

### Integration Logic

The fix is integrated into the existing registration flow:

```bash
# 1. Try standard wgcf register first
output=$(try_register)
ret=$?

# 2. If it fails with HTTP 429
if [[ "$output" == *"429"* || "$output" == *"Too Many Requests"* ]]; then
    # Use manual registration workaround
    if manual_warp_register; then
        # Success - wgcf-account.toml created
    fi
fi

# 3. Final fallback if still not registered
if [[ ! -f wgcf-account.toml ]]; then
    if manual_warp_register; then
        # Last attempt successful
    fi
fi
```

## Validation

The manual registration request has been tested and confirmed working:

```bash
# Generates valid registration
INSTALL_ID=$(tr -dc 'A-Za-z0-9' </dev/urandom | head -c 22)
FCM_TOKEN="${INSTALL_ID}:APA91b$(tr -dc 'A-Za-z0-9' </dev/urandom | head -c 134)"
TOS=$(date -u +"%Y-%m-%dT%H:%M:%S.000Z")
PUBKEY=$(wg genkey | wg pubkey)

curl -X POST 'https://api.cloudflareclient.com/v0a1922/reg' \
  -H 'User-Agent: okhttp/3.12.1' \
  -H 'CF-Client-Version: a-6.3-1922' \
  -H 'Content-Type: application/json' \
  --data "{
    \"fcm_token\":\"$FCM_TOKEN\",
    \"install_id\":\"$INSTALL_ID\",
    \"key\":\"$PUBKEY\",
    \"locale\":\"en_US\",
    \"model\":\"Android\",
    \"tos\":\"$TOS\",
    \"type\":\"Android\"
  }"

# Returns: HTTP/1.1 200 OK
```

## Compatibility

- **Backward compatible:** Still attempts `wgcf register` first
- **Graceful fallback:** Uses manual method only when needed
- **Same output format:** Creates identical `wgcf-account.toml` structure
- **Works with existing flow:** `wgcf generate` and subsequent steps unchanged

## Testing

To test the fix:

1. Remove existing account files:
   ```bash
   rm -f wgcf-account.toml wgcf-profile.conf
   ```

2. Run the installation script:
   ```bash
   bash install.sh
   ```

3. The script will:
   - Try `wgcf register` first
   - Detect HTTP 429 error
   - Automatically fall back to manual registration
   - Create `wgcf-account.toml` successfully
   - Continue with configuration generation

## Future Improvements

The **permanent fix** requires updating the `wgcf` source code itself:

**Repository:** https://github.com/ViRb3/wgcf

**File:** `cloudflare/api.go`

**Required changes:**
```go
RegisterRequest{
    FcmToken:  generateFcmToken(),     // Generate non-empty token
    InstallId: generateInstallId(),     // Generate non-empty ID  
    Key:       publicKey.String(),
    Locale:    "en_US",
    Model:     deviceModel,
    Tos:       time.Now().UTC().Format(time.RFC3339), // ISO-8601 format
    Type:      "Android",
}
```

Once `wgcf` is updated and a new release is published, this workaround can be removed.

## References

- Issue report: Documented HTTP 429 behavior with multiple VPS testing
- API endpoint: `https://api.cloudflareclient.com/v0a1922/reg`
- wgcf releases: https://github.com/ViRb3/wgcf/releases
- Current version in use: v2.2.32

## Credits

Fix implemented: 2026-09-09
Tested on: Ubuntu Linux (amd64)
