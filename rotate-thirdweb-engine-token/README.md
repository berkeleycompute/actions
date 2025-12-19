# Thirdweb Engine Access Token Rotation

This action rotates Thirdweb Engine access tokens by creating a new token and revoking the old one.

## API Reference

**Endpoint:** `POST /auth/access-tokens/create`  
**Documentation:** https://engine-api.thirdweb.com/reference#tag/access-tokens/POST/auth/access-tokens/create

## Action Location

- **Action:** `/actions/rotate-thirdweb-engine-token/action.yml`
- **Workflow:** `/devops/.github/workflows/rotate-thirdweb-engine-token.yml`

## Required Secrets

Configure these organization secrets:

```bash
# Thirdweb secret key (for authentication)
gh secret set THIRDWEB_SECRET_KEY --org berkeleycompute
# OR environment-specific:
gh secret set THIRDWEB_SECRET_KEY_TEST --org berkeleycompute
gh secret set THIRDWEB_SECRET_KEY_PROD --org berkeleycompute

# Current access tokens (will be rotated)
gh secret set THIRDWEB_ENGINE_ACCESS_TOKEN_TEST --org berkeleycompute
gh secret set THIRDWEB_ENGINE_ACCESS_TOKEN_PROD --org berkeleycompute

# GitHub App credentials (for updating org secrets)
gh secret set GH_APP_PRIVATE_KEY --org berkeleycompute
```

## Required Variables

Configure these organization variables:

```bash
# GitHub App configuration
gh variable set GH_APP_ID --body "<your-app-id>" --org berkeleycompute
gh variable set GH_APP_INSTALLATION_ID --body "<your-installation-id>" --org berkeleycompute

# Thirdweb Engine URLs
gh variable set THIRDWEB_ENGINE_URL_TEST --body "https://68ac0e06.engine-usw2.thirdweb.com" --org berkeleycompute
gh variable set THIRDWEB_ENGINE_URL_PROD --body "<your-prod-url>" --org berkeleycompute
```

## Usage

### Via GitHub Actions Workflow

1. Navigate to **Actions** → **Rotate Thirdweb Engine Access Token**
2. Click **Run workflow**
3. Select environment: `test` or `prod`
4. (Optional) Customize token label
5. Click **Run workflow**

The workflow will:
- Create a new access token
- Revoke the old token
- Update the organization secret with the new token

### Direct Action Usage

```yaml
- name: Rotate Thirdweb Engine token
  uses: ./actions/rotate-thirdweb-engine-token
  with:
    secret-key: ${{ secrets.THIRDWEB_SECRET_KEY }}
    engine-url: 'https://68ac0e06.engine-usw2.thirdweb.com'
    old-token: ${{ secrets.THIRDWEB_ENGINE_ACCESS_TOKEN_TEST }}
    label: 'GitHub Actions - Auto-rotated'
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `secret-key` | Thirdweb secret key for authentication | Yes | - |
| `engine-url` | Thirdweb Engine URL | Yes | - |
| `old-token` | Old access token to revoke | No | - |
| `label` | Label for the new access token | No | `GitHub Actions - Auto-rotated` |

## Outputs

| Output | Description |
|--------|-------------|
| `success` | Whether the token rotation was successful |
| `new-token` | The newly generated access token (masked in logs) |
| `token-id` | ID of the newly created token |

## Current Configuration

### Test Environment
- **Engine URL:** `https://68ac0e06.engine-usw2.thirdweb.com`
- **Access Token Secret:** `THIRDWEB_ENGINE_ACCESS_TOKEN_TEST` ✅ (Set)
- **Token Value:** `eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9...` (Set as organization secret)

### Production Environment
- **Engine URL:** Not configured (must be set via `THIRDWEB_ENGINE_URL_PROD` variable)
- **Access Token Secret:** `THIRDWEB_ENGINE_ACCESS_TOKEN_PROD` (Not set)

## How It Works

1. **Validation:** Checks that all required secrets and configurations are present
2. **Token Creation:** Calls the Thirdweb Engine API to create a new access token
3. **Token Revocation:** Revokes the old token by first listing all tokens, finding the old one, and revoking it
4. **Secret Update:** Uses GitHub App authentication to update the organization secret with the new token

## Security Notes

- The new token is masked in all logs
- Old tokens are revoked immediately after new token creation
- Requires GitHub App with 'Administration: Read and write' permission
- Uses secure token-based authentication throughout

## Troubleshooting

### Token creation fails
- Verify `THIRDWEB_SECRET_KEY` is set correctly
- Check that the Engine URL is correct and accessible
- Ensure the secret key has permission to create tokens

### Token revocation fails
- The new token will still be created successfully
- Old token may need to be manually revoked via the Thirdweb dashboard
- Check that the old token still exists and hasn't been revoked already

### Organization secret update fails
- Verify GitHub App credentials are correct
- Ensure the app has 'Administration: Read and write' permission
- Check that `GH_APP_ID` and `GH_APP_INSTALLATION_ID` are set correctly
- Manually update the secret if automatic update fails:
  ```bash
  gh secret set THIRDWEB_ENGINE_ACCESS_TOKEN_TEST --org berkeleycompute
  ```

## API Endpoints Used

1. **Create Token:** `POST /auth/access-tokens/create`
2. **List Tokens:** `GET /auth/access-tokens/get-all`
3. **Revoke Token:** `POST /auth/access-tokens/revoke`

## Next Steps

To complete the setup for production:

1. Set the production secret key:
   ```bash
   gh secret set THIRDWEB_SECRET_KEY_PROD --org berkeleycompute
   ```

2. Set the production Engine URL:
   ```bash
   gh variable set THIRDWEB_ENGINE_URL_PROD --body "<your-prod-url>" --org berkeleycompute
   ```

3. Set the current production access token:
   ```bash
   gh secret set THIRDWEB_ENGINE_ACCESS_TOKEN_PROD --org berkeleycompute
   ```

4. Run the workflow for test environment to verify it works correctly
