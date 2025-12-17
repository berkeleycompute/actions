# GitHub Actions Collection

Reusable GitHub Actions for managing GitHub App authentication, organization secrets, and AWS Secrets Manager.

## Quick Reference

| Action | Purpose | Use Case |
|--------|---------|----------|
| `get-jwt-token` | Generate GitHub App JWT | Authentication to GitHub API |
| `get-github-app-token` | Get installation token (JWT + access) | One-step GitHub App auth |
| `list-org-secrets` | List organization secrets | Audit or verify secrets |
| `update-org-secret` | Update GitHub org secret | Set specific secret values |
| `update-aws-secret` | Update AWS secret with custom value | Set AWS secrets to known values |
| `rotate-aws-secret` | Auto-generate and update AWS secret | Automated secret rotation |

### AWS Actions: When to Use Which?

- **Use `update-aws-secret`** when you need to set a secret to a **specific value** (e.g., API keys from external services, configuration values)
- **Use `rotate-aws-secret`** when you need to **generate a new random value** for security (e.g., database passwords, internal API keys, tokens)

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

### 5. `update-aws-secret`

Updates a secret in AWS Secrets Manager using environment-specific AWS credentials.

#### Inputs

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `environment` | ✅ Yes | - | Environment to target. Must be one of: `dev`, `devops`, `prod`, or `stage`. This parameter validates which environment you're updating and ensures you're using the correct credentials. |
| `aws-access-key-id` | ✅ Yes | - | AWS Access Key ID for the specified environment. Use organization secrets: `AWS_ACCESS_KEY_DEV`, `AWS_ACCESS_KEY_DEVOPS`, `AWS_ACCESS_KEY_PROD`, or `AWS_ACCESS_KEY_STAGE`. |
| `aws-secret-access-key` | ✅ Yes | - | AWS Secret Access Key for the specified environment. Use organization secrets: `AWS_SECRET_KEY_DEV`, `AWS_SECRET_KEY_DEVOPS`, `AWS_SECRET_KEY_PROD`, or `AWS_SECRET_KEY_STAGE`. |
| `aws-region` | ❌ No | `us-east-1` | AWS Region where the secret is stored (e.g., `us-west-2`, `us-east-1`, `eu-west-1`). |
| `secret-arn` | ✅ Yes | - | Full ARN of the AWS Secrets Manager secret to update. Format: `arn:aws:secretsmanager:REGION:ACCOUNT_ID:secret:SECRET_NAME-XXXXX` |
| `secret-value` | ✅ Yes | - | The new value to set for the secret. Can be plaintext or JSON string. For JSON secrets, pass a stringified JSON object. |

#### Outputs

| Output | Description |
|--------|-------------|
| `success` | Boolean indicating whether the secret was updated successfully (`true` or `false`) |
| `secret-arn` | The full ARN of the secret that was updated |
| `version-id` | The version ID assigned by AWS to this secret update |

#### Important Notes

1. **Environment Validation**: The action validates that `environment` matches one of the allowed values (`dev`, `devops`, `prod`, `stage`) and provides helpful error messages if mismatched credentials are used.

2. **Credential Matching**: Make sure the AWS credentials you pass match the environment. For example, if `environment: 'devops'`, you must use `AWS_ACCESS_KEY_DEVOPS` and `AWS_SECRET_KEY_DEVOPS`.

3. **Secret ARN Format**: The secret ARN must be the full ARN, not just the secret name. You can find this in the AWS Secrets Manager console or by running:
   ```bash
   aws secretsmanager describe-secret --secret-id YOUR_SECRET_NAME
   ```

4. **JSON Secrets**: If your secret stores JSON, pass it as a string:
   ```yaml
   secret-value: '{"key": "value", "another": "data"}'
   ```

#### Basic Example

```yaml
# Update a secret in the devops environment
- uses: berkeleycompute/actions/update-aws-secret@main
  id: update-aws-secret
  with:
    environment: 'devops'
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_DEVOPS }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_KEY_DEVOPS }}
    aws-region: 'us-west-2'
    secret-arn: 'arn:aws:secretsmanager:us-west-2:123456789012:secret:my-secret-abc123'
    secret-value: 'my-secret-value'
```

#### How to Use This Action

**Step 1: Identify your environment**
Determine which environment you're updating: `dev`, `devops`, `prod`, or `stage`.

**Step 2: Get the Secret ARN**
Find the ARN of your secret in AWS Secrets Manager. It looks like:
```
arn:aws:secretsmanager:us-west-2:123456789012:secret:my-secret-name-AbCdEf
```

**Step 3: Use the correct credentials**
Based on your environment, use the matching organization secrets:
- For `dev`: use `AWS_ACCESS_KEY_DEV` and `AWS_SECRET_KEY_DEV`
- For `devops`: use `AWS_ACCESS_KEY_DEVOPS` and `AWS_SECRET_KEY_DEVOPS`
- For `prod`: use `AWS_ACCESS_KEY_PROD` and `AWS_SECRET_KEY_PROD`
- For `stage`: use `AWS_ACCESS_KEY_STAGE` and `AWS_SECRET_KEY_STAGE`

**Step 4: Set the region**
Specify the AWS region where your secret is stored (defaults to `us-east-1` if not specified).

---

### 6. `rotate-aws-secret`

Automatically generates a new secure random value and updates an AWS Secrets Manager secret. Perfect for automated secret rotation.

#### Inputs

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `environment` | ✅ Yes | - | Environment to target. Must be one of: `dev`, `devops`, `prod`, or `stage`. |
| `aws-access-key-id` | ✅ Yes | - | AWS Access Key ID for the specified environment. Use organization secrets: `AWS_ACCESS_KEY_<ENV>`. |
| `aws-secret-access-key` | ✅ Yes | - | AWS Secret Access Key for the specified environment. Use organization secrets: `AWS_SECRET_KEY_<ENV>`. |
| `aws-region` | ❌ No | `us-east-1` | AWS Region where the secret is stored. |
| `secret-arn` | ✅ Yes | - | Full ARN of the AWS Secrets Manager secret to rotate. |
| `secret-length` | ❌ No | `32` | Length of the generated secret (minimum: 8, maximum: 256). |
| `secret-type` | ❌ No | `alphanumeric` | Type of secret to generate: `alphanumeric`, `base64`, or `hex`. |
| `include-special-chars` | ❌ No | `true` | Include special characters in alphanumeric secrets (`true` or `false`). |

#### Outputs

| Output | Description |
|--------|-------------|
| `success` | Boolean indicating whether the secret was rotated successfully |
| `secret-arn` | The full ARN of the secret that was rotated |
| `version-id` | The version ID assigned by AWS to this secret rotation |
| `new-secret-value` | The newly generated secret value (masked in logs, use to update dependent services) |

#### Secret Types

- **`alphanumeric`**: Random letters, numbers, and optionally special characters (`!@#$%^&*()_+-=[]{}|;:,.<>?`)
- **`base64`**: Base64-encoded random bytes
- **`hex`**: Hexadecimal string (0-9, a-f)

#### Basic Example

```yaml
# Rotate a database password in the prod environment
- uses: berkeleycompute/actions/rotate-aws-secret@main
  id: rotate-db-password
  with:
    environment: 'prod'
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_PROD }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_KEY_PROD }}
    aws-region: 'us-west-2'
    secret-arn: 'arn:aws:secretsmanager:us-west-2:123456789012:secret:db-password-abc123'
    secret-length: '32'
    secret-type: 'alphanumeric'
    include-special-chars: 'true'

- name: Use the new secret
  run: |
    echo "Secret rotated successfully!"
    echo "New version: ${{ steps.rotate-db-password.outputs.version-id }}"
    # The new secret value is in: ${{ steps.rotate-db-password.outputs.new-secret-value }}
```

#### Complete Rotation Example

```yaml
name: Rotate Production Database Password

on:
  schedule:
    - cron: '0 2 * * 0'  # Every Sunday at 2 AM UTC
  workflow_dispatch:

jobs:
  rotate-password:
    runs-on: ubuntu-latest
    steps:
      - name: Rotate AWS secret
        id: rotate
        uses: berkeleycompute/actions/rotate-aws-secret@main
        with:
          environment: 'prod'
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_PROD }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_KEY_PROD }}
          aws-region: 'us-east-1'
          secret-arn: ${{ secrets.PROD_DB_PASSWORD_ARN }}
          secret-length: '48'
          secret-type: 'alphanumeric'
      
      - name: Update database with new password
        run: |
          # Use the new password to update your database
          # The value is available in: ${{ steps.rotate.outputs.new-secret-value }}
          echo "Password rotated to version: ${{ steps.rotate.outputs.version-id }}"
      
      - name: Notify team
        if: success()
        run: echo "✅ Database password rotated successfully"
```

#### Important Notes

1. **Automatic Generation**: The action generates a cryptographically secure random value - you don't need to provide one.

2. **New Secret in Output**: The newly generated secret is available in the `new-secret-value` output. Use this to update other services that depend on this secret.

3. **Minimum Length**: For security, the minimum secret length is 8 characters. Recommended: 32+ for passwords, 64+ for API keys.

4. **Secret Types Guide**:
   - Use `alphanumeric` with special chars for passwords
   - Use `hex` for API keys and tokens
   - Use `base64` for binary data or when you need maximum entropy

## Complete Examples

### Example 1: Manage GitHub Organization Secrets

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

### Example 2: Update AWS Secrets Manager Secret

```yaml
name: Update AWS Secret

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment (dev, devops, prod, stage)'
        required: true
        type: choice
        options:
          - dev
          - devops
          - prod
          - stage
      secret-value:
        description: 'New secret value'
        required: true

jobs:
  update-aws-secret:
    runs-on: ubuntu-latest
    steps:
      - name: Set environment-specific secrets
        id: set-secrets
        run: |
          ENV_UPPER=$(echo "${{ github.event.inputs.environment }}" | tr '[:lower:]' '[:upper:]')
          echo "env-upper=$ENV_UPPER" >> $GITHUB_OUTPUT
      
      - uses: berkeleycompute/actions/update-aws-secret@main
        id: update-secret
        with:
          environment: ${{ github.event.inputs.environment }}
          aws-access-key-id: ${{ secrets[format('AWS_ACCESS_KEY_{0}', steps.set-secrets.outputs.env-upper)] }}
          aws-secret-access-key: ${{ secrets[format('AWS_SECRET_KEY_{0}', steps.set-secrets.outputs.env-upper)] }}
          aws-region: 'us-west-2'
          secret-arn: 'arn:aws:secretsmanager:us-west-2:123456789012:secret:my-secret-abc123'
          secret-value: ${{ github.event.inputs.secret-value }}
      
      - name: Check result
        run: |
          echo "Secret updated: ${{ steps.update-secret.outputs.success }}"
          echo "Version ID: ${{ steps.update-secret.outputs.version-id }}"
```

### Example 3: Update AWS Secret for Specific Environment

```yaml
name: Update DevOps AWS Secret

on:
  workflow_dispatch:
    inputs:
      secret-value:
        description: 'New secret value'
        required: true

jobs:
  update-devops-secret:
    runs-on: ubuntu-latest
    steps:
      - uses: berkeleycompute/actions/update-aws-secret@main
        with:
          environment: 'devops'
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_DEVOPS }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_KEY_DEVOPS }}
          aws-region: 'us-east-1'
          secret-arn: 'arn:aws:secretsmanager:us-east-1:123456789012:secret:devops-secret-abc123'
          secret-value: ${{ github.event.inputs.secret-value }}
```

### Example 4: Complete AWS Secret Management Workflow

```yaml
name: Manage AWS Secrets

on:
  workflow_dispatch:
    inputs:
      action:
        description: 'Action to perform'
        type: choice
        required: true
        options:
          - rotate-password
          - update-api-key
      environment:
        description: 'Environment'
        type: choice
        required: true
        options:
          - dev
          - devops
          - prod
          - stage
      api-key:
        description: 'API Key (only for update-api-key action)'
        required: false

jobs:
  manage-secrets:
    runs-on: ubuntu-latest
    steps:
      - name: Rotate database password
        if: github.event.inputs.action == 'rotate-password'
        id: rotate
        uses: berkeleycompute/actions/rotate-aws-secret@main
        with:
          environment: ${{ github.event.inputs.environment }}
          aws-access-key-id: ${{ secrets[format('AWS_ACCESS_KEY_{0}', github.event.inputs.environment)] }}
          aws-secret-access-key: ${{ secrets[format('AWS_SECRET_KEY_{0}', github.event.inputs.environment)] }}
          aws-region: 'us-west-2'
          secret-arn: 'arn:aws:secretsmanager:us-west-2:123456789012:secret:db-password-abc123'
          secret-length: '48'
      
      - name: Update API key
        if: github.event.inputs.action == 'update-api-key'
        id: update
        uses: berkeleycompute/actions/update-aws-secret@main
        with:
          environment: ${{ github.event.inputs.environment }}
          aws-access-key-id: ${{ secrets[format('AWS_ACCESS_KEY_{0}', github.event.inputs.environment)] }}
          aws-secret-access-key: ${{ secrets[format('AWS_SECRET_KEY_{0}', github.event.inputs.environment)] }}
          aws-region: 'us-west-2'
          secret-arn: 'arn:aws:secretsmanager:us-west-2:123456789012:secret:api-key-xyz789'
          secret-value: ${{ github.event.inputs.api-key }}
      
      - name: Report results
        run: |
          if [ "${{ github.event.inputs.action }}" = "rotate-password" ]; then
            echo "✅ Password rotated successfully"
            echo "Version ID: ${{ steps.rotate.outputs.version-id }}"
          else
            echo "✅ API key updated successfully"
            echo "Version ID: ${{ steps.update.outputs.version-id }}"
          fi
```

## Requirements

### For GitHub Actions (get-jwt-token, get-installation-token, list-org-secrets, update-org-secret)

#### Organization Secrets

These secrets must be set in your organization:

1. **GH_APP_ID**: The GitHub App ID
2. **GH_APP_PRIVATE_KEY**: The private key (PEM file contents) for the GitHub App
3. **GH_APP_INSTALLATION_ID**: The installation ID

#### GitHub App Permissions

The GitHub App must have:
- **Organization permissions**: `Administration: Read and write`

### For AWS Actions (update-aws-secret)

#### Required Secrets

Environment-specific AWS credentials (where `<ENV>` is `DEV`, `DEVOPS`, `PROD`, or `STAGE`):

1. **AWS_ACCESS_KEY_DEV**: AWS Access Key ID for dev environment
2. **AWS_SECRET_KEY_DEV**: AWS Secret Access Key for dev environment
3. **AWS_ACCESS_KEY_DEVOPS**: AWS Access Key ID for devops environment
4. **AWS_SECRET_KEY_DEVOPS**: AWS Secret Access Key for devops environment
5. **AWS_ACCESS_KEY_PROD**: AWS Access Key ID for prod environment
6. **AWS_SECRET_KEY_PROD**: AWS Secret Access Key for prod environment
7. **AWS_ACCESS_KEY_STAGE**: AWS Access Key ID for stage environment
8. **AWS_SECRET_KEY_STAGE**: AWS Secret Access Key for stage environment

#### IAM Permissions

The AWS credentials must have the following IAM permissions:
- `secretsmanager:PutSecretValue` - to update secret values
- `secretsmanager:DescribeSecret` - to verify the secret exists (optional but recommended)

## Usage in Other Repositories

To use these actions in other repositories:

```yaml
- uses: berkeleycompute/actions/get-jwt-token@main
  with:
    app-id: ${{ secrets.GH_APP_ID }}
    private-key: ${{ secrets.GH_APP_PRIVATE_KEY }}
```

Make sure the required organization secrets are available in the repository or organization where you're using these actions.

