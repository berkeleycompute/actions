# GitHub App Actions

Reusable GitHub Actions for managing GitHub App authentication and organization secrets.

## Available Actions

### 1. `get-jwt-token`

Generates a JWT token for GitHub App authentication.

**Inputs:**
- `app-id` (required): GitHub App ID
- `private-key` (required): GitHub App private key (PEM format)

**Outputs:**
- `jwt-token`: Generated JWT token

**Example:**
```yaml
- uses: berkeleycompute/actions/get-jwt-token@main
  id: get-jwt
  with:
    app-id: ${{ secrets.GH_APP_ID }}
    private-key: ${{ secrets.GH_APP_PRIVATE_KEY }}
```

### 2. `get-installation-token`

Gets an installation access token from a JWT token.

**Inputs:**
- `jwt-token` (required): JWT token from `get-jwt-token` action
- `installation-id` (required): GitHub App installation ID

**Outputs:**
- `token`: Installation access token
- `expires-at`: Token expiration timestamp

**Example:**
```yaml
- uses: berkeleycompute/actions/get-installation-token@main
  id: get-token
  with:
    jwt-token: ${{ steps.get-jwt.outputs.jwt-token }}
    installation-id: ${{ secrets.GH_APP_INSTALLATION_ID }}
```

### 3. `list-org-secrets`

Lists all organization secrets.

**Inputs:**
- `token` (required): Installation access token
- `organization` (required): Organization name

**Outputs:**
- `secrets-list`: JSON array of organization secrets
- `secrets-count`: Number of secrets found

**Example:**
```yaml
- uses: berkeleycompute/actions/list-org-secrets@main
  id: list-secrets
  with:
    token: ${{ steps.get-token.outputs.token }}
    organization: ${{ github.repository_owner }}
```

### 4. `update-org-secret`

Updates or creates an organization secret.

**Inputs:**
- `token` (required): Installation access token
- `organization` (required): Organization name
- `secret-name` (required): Name of the secret to update
- `secret-value` (required): Value to set for the secret

**Outputs:**
- `success`: Whether the secret was updated successfully
- `secret-name`: Name of the secret that was updated

**Example:**
```yaml
- uses: berkeleycompute/actions/update-org-secret@main
  id: update-secret
  with:
    token: ${{ steps.get-token.outputs.token }}
    organization: ${{ github.repository_owner }}
    secret-name: 'MY_SECRET'
    secret-value: 'my-secret-value'
```

## Complete Example

```yaml
name: Manage Organization Secrets

on:
  workflow_dispatch:

jobs:
  manage-secrets:
    runs-on: ubuntu-latest
    steps:
      # Step 1: Get JWT token
      - uses: berkeleycompute/actions/get-jwt-token@main
        id: get-jwt
        with:
          app-id: ${{ secrets.GH_APP_ID }}
          private-key: ${{ secrets.GH_APP_PRIVATE_KEY }}

      # Step 2: Get installation access token
      - uses: berkeleycompute/actions/get-installation-token@main
        id: get-token
        with:
          jwt-token: ${{ steps.get-jwt.outputs.jwt-token }}
          installation-id: ${{ secrets.GH_APP_INSTALLATION_ID }}

      # Step 3: List secrets
      - uses: berkeleycompute/actions/list-org-secrets@main
        id: list-secrets
        with:
          token: ${{ steps.get-token.outputs.token }}
          organization: ${{ github.repository_owner }}

      # Step 4: Update a secret
      - uses: berkeleycompute/actions/update-org-secret@main
        id: update-secret
        with:
          token: ${{ steps.get-token.outputs.token }}
          organization: ${{ github.repository_owner }}
          secret-name: 'MY_SECRET'
          secret-value: 'my-new-value'
```

## Requirements

### Organization Secrets

These secrets must be set in your organization:

1. **GH_APP_ID**: The GitHub App ID
2. **GH_APP_PRIVATE_KEY**: The private key (PEM file contents) for the GitHub App
3. **GH_APP_INSTALLATION_ID**: The installation ID

### GitHub App Permissions

The GitHub App must have:
- **Organization permissions**: `Administration: Read and write`

## Usage in Other Repositories

To use these actions in other repositories:

```yaml
- uses: berkeleycompute/actions/get-jwt-token@main
  with:
    app-id: ${{ secrets.GH_APP_ID }}
    private-key: ${{ secrets.GH_APP_PRIVATE_KEY }}
```

Make sure the required organization secrets are available in the repository or organization where you're using these actions.

