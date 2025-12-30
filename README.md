# GitHub Actions Collection

Reusable GitHub Actions for managing GitHub App authentication, organization secrets, AWS Secrets Manager, AWS IAM access keys, and Supabase database passwords.

## Quick Reference

| Action | Purpose | Use Case |
|--------|---------|----------|
| `get-jwt-token` | Generate GitHub App JWT | Authentication to GitHub API |
| `get-github-app-token` | Get installation token (JWT + access) | One-step GitHub App auth |
| `list-org-secrets` | List organization secrets | Audit or verify secrets |
| `update-org-secret` | Update GitHub org secret | Set specific secret values |
| `update-aws-secret` | Update AWS secret with custom value | Set AWS secrets to known values |
| `rotate-aws-secret` | Auto-generate and update AWS secret | Automated secret rotation |
| `rotate-aws-access-key` | Rotate AWS IAM access keys | Rotate IAM user access keys |
| `rotate-supabase-db-password` | Auto-generate and update Supabase DB password | Automated Supabase password rotation |

### AWS Actions: When to Use Which?

- **Use `update-aws-secret`** when you need to set a secret to a **specific value** (e.g., API keys from external services, configuration values)
- **Use `rotate-aws-secret`** when you need to **generate a new random value** for security (e.g., database passwords, internal API keys, tokens)
- **Use `rotate-aws-access-key`** when you need to **rotate IAM user access keys** (creates new key and deletes old one)

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

---

### 7. `rotate-aws-access-key`

Rotates AWS IAM access keys by creating a new access key and deleting the old one. This is useful for security best practices and automated key rotation.

#### Inputs

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `environment` | ✅ Yes | - | Environment to target. Must be one of: `dev`, `devops`, `prod`, or `stage`. |
| `aws-access-key-id` | ✅ Yes | - | AWS Access Key ID for the specified environment. Use organization secrets: `AWS_ACCESS_KEY_<ENV>`. |
| `aws-secret-access-key` | ✅ Yes | - | AWS Secret Access Key for the specified environment. Use organization secrets: `AWS_SECRET_KEY_<ENV>`. |
| `aws-region` | ❌ No | `us-east-1` | AWS Region where the IAM user is located. |
| `iam-username` | ✅ Yes | - | IAM username whose access key will be rotated. |

#### Outputs

| Output | Description |
|--------|-------------|
| `success` | Boolean indicating whether the access key was rotated successfully |
| `new-access-key-id` | The newly created access key ID (masked in logs) |
| `new-secret-access-key` | The newly created secret access key (masked in logs) |
| `old-access-key-id` | The old access key ID that was deleted |

#### Basic Example

```yaml
# Rotate an IAM user's access key
- uses: berkeleycompute/actions/rotate-aws-access-key@main
  id: rotate-access-key
  with:
    environment: 'devops'
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_DEVOPS }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_KEY_DEVOPS }}
    aws-region: 'us-east-1'
    iam-username: 'my-iam-user'

- name: Use the new access key
  run: |
    echo "Access key rotated successfully!"
    echo "New Access Key ID: ${{ steps.rotate-access-key.outputs.new-access-key-id }}"
    # The new secret access key is in: ${{ steps.rotate-access-key.outputs.new-secret-access-key }}
```

#### Complete Rotation Example

```yaml
name: Rotate IAM Access Key

on:
  schedule:
    - cron: '0 2 * * 0'  # Every Sunday at 2 AM UTC
  workflow_dispatch:
    inputs:
      iam-username:
        description: 'IAM username to rotate'
        required: true

jobs:
  rotate-key:
    runs-on: ubuntu-latest
    steps:
      - name: Rotate AWS access key
        id: rotate
        uses: berkeleycompute/actions/rotate-aws-access-key@main
        with:
          environment: 'prod'
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_PROD }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_KEY_PROD }}
          aws-region: 'us-east-1'
          iam-username: ${{ github.event.inputs.iam-username }}
      
      - name: Update GitHub secrets with new key
        run: |
          # Use the new access key to update GitHub secrets or other services
          echo "Old key deleted: ${{ steps.rotate.outputs.old-access-key-id }}"
          echo "New key created: ${{ steps.rotate.outputs.new-access-key-id }}"
          # The new secret access key is in: ${{ steps.rotate.outputs.new-secret-access-key }}
      
      - name: Notify team
        if: success()
        run: echo "✅ IAM access key rotated successfully for ${{ github.event.inputs.iam-username }}"
```

#### Important Notes

1. **AWS Key Limit**: Each IAM user can have a maximum of 2 access keys. If the user already has 2 keys, you must delete one manually before running this action.

2. **Current Key Usage**: If the access key being used for authentication is the one being rotated, the action will still work, but ensure you save the new key before the old one is deleted.

3. **New Keys in Output**: The newly created access key ID and secret access key are available in the outputs. **Save these immediately** as the secret access key is only shown once.

4. **Update Dependencies**: After rotation, update all services, applications, and configurations that use the old access key with the new credentials.

5. **IAM Permissions**: The AWS credentials used must have the following IAM permissions:
   - `iam:ListAccessKeys` - to list existing keys
   - `iam:CreateAccessKey` - to create new keys (intended for CLI/API usage)
   - `iam:DeleteAccessKey` - to delete old keys
   - `sts:GetCallerIdentity` - to verify new keys work (used during rotation)

7. **Error Handling**: The action follows a safe rotation process:
   - Creates new key first
   - Verifies new key works with AWS STS
   - Only deletes old key if verification succeeds
   - If verification fails, old key remains active and new key is preserved

---

### 8. `rotate-supabase-db-password`

Automatically generates a new secure random password and updates the Supabase database password via the Management API. Perfect for automated password rotation.

#### Inputs

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `supabase-access-token` | ✅ Yes | - | Supabase Management API access token. Get this from your Supabase dashboard under Settings > Access Tokens. |
| `project-ref` | ✅ Yes | - | Supabase project reference ID. Found in your project's URL: `https://supabase.com/dashboard/project/<project-ref>`. |
| `password-length` | ❌ No | `32` | Length of the generated password (minimum: 12, maximum: 128). Recommended: 32+ for production. |
| `include-special-chars` | ❌ No | `true` | Include special characters in password (`true` or `false`). |
| `supabase-api-url` | ❌ No | `https://api.supabase.com` | Supabase Management API base URL. Usually only needs to be changed for self-hosted instances. |

#### Outputs

| Output | Description |
|--------|-------------|
| `success` | Boolean indicating whether the password was rotated successfully |
| `project-ref` | The project reference that was updated |
| `new-password` | The newly generated password (masked in logs, use to update dependent services) |

#### Basic Example

```yaml
# Rotate a Supabase database password
- uses: berkeleycompute/actions/rotate-supabase-db-password@main
  id: rotate-supabase-password
  with:
    supabase-access-token: ${{ secrets.SUPABASE_ACCESS_TOKEN }}
    project-ref: 'abcdefghijklmnop'
    password-length: '32'
    include-special-chars: 'true'

- name: Use the new password
  run: |
    echo "Password rotated successfully!"
    # The new password is in: ${{ steps.rotate-supabase-password.outputs.new-password }}
```

#### Complete Rotation Example

```yaml
name: Rotate Supabase Database Password

on:
  schedule:
    - cron: '0 2 * * 0'  # Every Sunday at 2 AM UTC
  workflow_dispatch:

jobs:
  rotate-password:
    runs-on: ubuntu-latest
    steps:
      - name: Rotate Supabase database password
        id: rotate
        uses: berkeleycompute/actions/rotate-supabase-db-password@main
        with:
          supabase-access-token: ${{ secrets.SUPABASE_ACCESS_TOKEN }}
          project-ref: ${{ secrets.SUPABASE_PROJECT_REF }}
          password-length: '48'
          include-special-chars: 'true'
      
      - name: Update connection strings
        run: |
          # Use the new password to update your application's connection strings
          # The value is available in: ${{ steps.rotate.outputs.new-password }}
          echo "Password rotated for project: ${{ steps.rotate.outputs.project-ref }}"
      
      - name: Notify team
        if: success()
        run: echo "✅ Supabase database password rotated successfully"
```

#### Important Notes

1. **Automatic Generation**: The action generates a cryptographically secure random password - you don't need to provide one.

2. **Connection Termination**: Resetting the database password will terminate all existing database connections. Ensure you update all applications and services that connect to this database with the new password.

3. **Minimum Length**: For security, the minimum password length is 12 characters. Recommended: 32+ for production databases.

4. **Access Token Permissions**: The Supabase access token must have permissions to update database passwords. This typically requires the token to have project management permissions.

5. **New Password in Output**: The newly generated password is available in the `new-password` output. Use this to update other services that depend on this password (e.g., connection strings, environment variables, secrets managers).

6. **Finding Your Project Reference**: 
   - Go to your Supabase project dashboard
   - The project reference is in the URL: `https://supabase.com/dashboard/project/<project-ref>`
   - Or find it in Project Settings > General

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

### For AWS Actions (update-aws-secret, rotate-aws-secret, rotate-aws-access-key)

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

**For `update-aws-secret` and `rotate-aws-secret`:**
- `secretsmanager:PutSecretValue` - to update secret values
- `secretsmanager:DescribeSecret` - to verify the secret exists (optional but recommended)

**For `rotate-aws-access-key`:**
- `iam:ListAccessKeys` - to list existing access keys for a user
- `iam:CreateAccessKey` - to create new access keys (with CLI use case)
- `iam:DeleteAccessKey` - to delete old access keys
- `iam:GetUser` - to verify if user exists (optional, for user verification)
- `iam:CreateUser` - to create user if it doesn't exist (optional, for automated setup)
- `iam:TagUser` - to tag created users (optional)
- `sts:GetCallerIdentity` - to verify new access keys work (used during rotation)

### For Supabase Actions (rotate-supabase-db-password)

#### Required Secrets

1. **SUPABASE_ACCESS_TOKEN**: Supabase Management API access token
   - Get this from your Supabase dashboard: Settings > Access Tokens
   - Create a new token with appropriate permissions if needed
   - The token must have permissions to update database passwords

2. **SUPABASE_PROJECT_REF**: Supabase project reference ID (optional, can be passed directly)
   - Found in your project's URL: `https://supabase.com/dashboard/project/<project-ref>`
   - Or in Project Settings > General

#### API Permissions

The Supabase access token must have:
- **Project management permissions** - to update database passwords
- Access to the specific project you want to rotate the password for

#### Getting Your Access Token

1. Log in to your Supabase dashboard
2. Navigate to Settings > Access Tokens
3. Create a new access token or use an existing one
4. Store it securely in your GitHub secrets

## Usage in Other Repositories

To use these actions in other repositories:

```yaml
- uses: berkeleycompute/actions/get-jwt-token@main
  with:
    app-id: ${{ secrets.GH_APP_ID }}
    private-key: ${{ secrets.GH_APP_PRIVATE_KEY }}
```

Make sure the required organization secrets are available in the repository or organization where you're using these actions.

