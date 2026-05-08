# Tauri Signing Secrets Configuration

This project uses Tauri's updater feature for automatic updates. To enable this, you need to configure GitHub Secrets with the following:

## Required Secrets

### 1. `TAURI_SIGNING_PRIVATE_KEY`
The RSA private key (PEM format) used to sign updater artifacts.

**How to add:**
1. Go to GitHub → Your repository → Settings → Secrets and variables → Actions
2. Click "New repository secret"
3. Add the following secrets:

```
Name: TAURI_SIGNING_PRIVATE_KEY
Value: (paste the contents of tauri/src-tauri/tauri-signing-key.pem)
```

### 2. `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` (Optional for v1Compatible)
If you're using `createUpdaterArtifacts: "v1Compatible"`, you need to provide the password for the private key.

**How to add:**
```
Name: TAURI_SIGNING_PRIVATE_KEY_PASSWORD  
Value: (the password you set when generating the key)
```

## Generating a New Key Pair

If you don't have a key pair, generate one:

```bash
cd tauri/src-tauri

# Generate RSA private key (4096 bits)
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 \
  -out tauri-signing-key.pem

# Set a password for the key (important!)
openssl pkey -in tauri-signing-key.pem \
  -passout pass:YourStrongPassword123 \
  -out tauri-signing-key-passworded.pem

# Generate public key (no password needed)
openssl pkey -in tauri-signing-key-passworded.pem \
  -pubout -out tauri-signing-key.pub

# Convert public key to base64 for tauri.conf.json
base64 -i tauri-signing-key.pub > tauri-signing-pubkey.b64
```

## Updating `tauri.conf.json`

After generating the public key, update `tauri/src-tauri/tauri.conf.json`:

```json
{
  "plugins": {
    "updater": {
      "pubkey": "<base64-encoded-public-key-from-above>",
      // ... other config
    }
  }
}
```

## Choosing the Right Mode

### v1Compatible (requires password)
- Use when: You want full updater functionality with signature verification
- Requires: `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` secret to be configured in GitHub Secrets
- Config: `createUpdaterArtifacts: "v1Compatible"`

### No Updater (no password needed)
- Use when: You don't have a private key password configured in CI/CD, or you're not using the updater feature
- Config: `createUpdaterArtifacts: "false"` or `"none"`

## Current Configuration

This repository currently uses **No Updater** mode for Windows builds:

```yaml
createUpdaterArtifacts: "false"  # Changed from "v1Compatible"
```

This is because the Windows CI workflow doesn't have `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` configured.

## Enabling Updater in CI/CD (Optional)

If you want to enable automatic updates with signature verification:

### Option 1: Use an unpassworded key (less secure)
```bash
# Generate without password
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 \
  -out tauri/src-tauri/tauri-signing-key.pem
```

Then in GitHub Secrets:
- Add `TAURI_SIGNING_PRIVATE_KEY` with the key contents
- Do NOT add `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` (leave it empty or unset)

### Option 2: Use a passworded key (recommended for production)
```bash
# Generate with a strong password
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 \
  -out tauri/src-tauri/tauri-signing-key.pem

# Add password protection
openssl pkey -in tauri/src-tauri/tauri-signing-key.pem \
  -passout pass:YourVeryStrongPassword123! \
  -out tauri/src-tauri/tauri-signing-key-passworded.pem
```

Then in GitHub Secrets:
- Add `TAURI_SIGNING_PRIVATE_KEY` with the contents of `tauri-signing-key-passworded.pem`
- Add `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` with the password you used

## Security Notes

- **Never commit private keys to the repository**
  - The `.gitignore` now excludes `tauri-signing-key.pem` and `tauri-signing-key-passworded.pem`
  - The generated keys in this repo are for local development only
- Keep your key password strong and secure (at least 12 characters with mixed case, numbers, and symbols)
- Rotate keys periodically (revoke old ones, generate new ones)
- If you lose your private key, you'll need to regenerate it and update `tauri.conf.json` with the new public key

## For Production Deployment

When you're ready to deploy with automatic updates:

1. Generate a new key pair with a password
2. Add both `TAURI_SIGNING_PRIVATE_KEY` and `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` to GitHub Secrets
3. Update `tauri.conf.json` with the new public key (base64 encoded)
4. Change `createUpdaterArtifacts` to `"v1Compatible"` in your CI workflows

## Security Notes

- **Never commit private keys to the repository**
- The `tauri-signing-key.pem` file in this repo is for local development only
- Keep your key password strong and secure
- Rotate keys periodically (revoke old ones, generate new ones)
