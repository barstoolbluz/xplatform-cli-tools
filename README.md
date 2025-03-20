# Secure Credentials Management for CLI Workflows with Flox and 1Password 🔐

This Flox environment provides a secure way to manage credentials for common developer tools by integrating with 1Password. It prevents credentials from being stored in unencrypted files on disk, significantly reducing the risk of credential leakage. It supports both local development and CI/CD environments.

## Installed Packages

- **1Password CLI** (`op`): Used for secure credential management
- **AWS CLI 2** (`aws`): For interacting with AWS services
- **GitHub CLI** (`gh`): For interacting with GitHub repositories
- **Git** (full version): For version control
- **Gum**: A tool for glamorous shell scripts
- **Bat**: A cat clone with syntax highlighting and Git integration
- **Curl**: Command-line tool for transferring data with URLs

## Security Benefits ✅

Many development tools store credentials in unencrypted files:
- GitHub CLI stores tokens in `~/.config/gh/hosts.yml`
- AWS CLI stores credentials in `~/.aws/credentials`
- Git may cache credentials in plaintext in some configurations

This environment wraps these tools to avoid persistent credential storage by:
1. Fetching credentials from 1Password at runtime
2. Injecting them into commands via environment variables in ephemeral subshells
3. Ensuring credentials are never written to disk and exist only for the duration of the command

## How It Works

This environment implements wrapper functions for `git`, `gh`, and `aws` that:

1. Extract credentials from 1Password at runtime
2. Pass these credentials to the underlying commands securely
3. Clean up any temporary files after execution

Credentials are available only for the duration of the command and never written to unencrypted files.

### Authentication Methods

The wrapper functions use two different approaches for credential handling:

#### `op run` (for `gh` and `aws`)
- Used by the `gh` and `aws` wrappers
- Executes commands in an ephemeral subshell
- Retrieves secrets directly from 1Password and exports them as environment variables
- Credentials exist only within this ephemeral subshell
- When the command finishes executing, the subshell is destroyed, along with any credentials

#### `op read` (for `git`)
- Used by the `git` wrapper for operations requiring authentication
- Directly reads the token from 1Password
- Creates a temporary script (via `GIT_ASKPASS`) that outputs the token when Git requests it
- The token is never written to disk in plaintext (only to a temporary file that is immediately deleted)
- Benefits from the process isolation of the subshell, though this is not as strong as container isolation
- The temporary file and token are cleaned up when the command completes using trap-based cleanup mechanisms specific to each shell

Both methods ensure credentials are never persistently stored in unencrypted files and exist only for the duration needed to complete the command.

### Environment-Aware Authentication Flow

The environment now intelligently detects whether it's running in a local development environment or a CI/CD environment and adapts its authentication method accordingly:

#### Local Development Environment
1. On environment activation, you'll authenticate with 1Password interactively
2. Your 1Password session token is stored both in memory and in a local file for persistence
3. When you run a wrapped command, it fetches the required credentials from 1Password
4. The credentials exist only for the duration of the command execution
5. If authentication fails, you'll get up to 3 retry attempts

#### CI/CD Environment
1. The environment detects that it's running in CI (currently supports GitHub Actions)
2. Authentication occurs non-interactively using 1Password service accounts
3. The 1Password session token is kept exclusively in environment variables (no file I/O)
4. Wrapper functions prioritize environment variables for authentication
5. All operations remain purely in memory to eliminate potential file-based failure points

## 1Password Vault Configuration

### Local Development Setup

Once you have the 1Password CLI set up (see **Prerequisites**, below), you'll need to customize the environment variables to match your specific 1Password vault structure. The default values in the `manifest.toml` are examples and will need to be modified:

```toml
[vars]
# 1password github config
OP_GITHUB_VAULT = "vault"           # Name of 1Password vault containing GitHub tokens
OP_GITHUB_TOKEN_ITEM = "token_item"           # Name of the item storing GitHub token
OP_GITHUB_TOKEN_FIELD = "token_field"         # Field name containing the GitHub token

# 1password aws config
OP_AWS_VAULT = "vault"              # Name of 1Password vault containing AWS credentials
OP_AWS_CREDENTIALS_ITEM = "aws_creds"     # Name of the item storing AWS credentials
OP_AWS_USERNAME_FIELD = "username"      # Field name for AWS access key ID
OP_AWS_CREDENTIALS_FIELD = "credentials" # Field name for AWS secret access key
```

**Important:** You must modify these environment variables to match your own 1Password vault structure:

1. **For GitHub access**: 
   - Set `OP_GITHUB_VAULT` to the name of your vault containing GitHub tokens
   - Set `OP_GITHUB_TOKEN_ITEM` to the name of your item storing the GitHub token
   - Set `OP_GITHUB_TOKEN_FIELD` to the field name containing your GitHub token

2. **For AWS access**:
   - Set `OP_AWS_VAULT` to the name of your vault containing AWS credentials
   - Set `OP_AWS_CREDENTIALS_ITEM` to the name of your item storing AWS credentials
   - Set `OP_AWS_USERNAME_FIELD` to the field name for your AWS access key ID
   - Set `OP_AWS_CREDENTIALS_FIELD` to the field name for your AWS secret access key

### CI/CD Environment Setup

For CI/CD environments, you'll need to set up 1Password service accounts and configure the appropriate environment variables:

1. **Create a 1Password Service Account**:
   - Follow [1Password's documentation](https://developer.1password.com/docs/service-accounts) to create a service account
   - Generate a service account token with appropriate access to your vaults

2. **Configure GitHub Actions**:
   - Add the service account token as a secret in your GitHub repository:
     ```
     OP_SERVICE_ACCOUNT_TOKEN=your-service-account-token
     ```
   - Ensure any other required configuration variables are also set as GitHub secrets

3. **GitHub Actions Workflow Example**:
   ```yaml
   name: CI with 1Password Integration
   
   on:
     push:
       branches: [ main ]
   
   jobs:
     build:
       runs-on: ubuntu-latest
       
       steps:
       - uses: actions/checkout@v3
       
       - name: Set up Flox
         uses: flox-examples/setup-flox@v1
       
       - name: Activate environment with credentials
         env:
           OP_SERVICE_ACCOUNT_TOKEN: ${{ secrets.OP_SERVICE_ACCOUNT_TOKEN }}
         run: flox activate
       
       # Your other steps that use the secure credentials
   ```

The environment will automatically detect that it's running in GitHub Actions and use the service account token for non-interactive authentication.

## Prerequisites

### 1Password CLI Setup (Local Development)

This environment expects the 1Password CLI to be already set up on your system. Specifically, it looks for a config file at `~/.config/op/config`.

#### Option 1: Automatic Setup

You can use the provided wizard to set up 1Password CLI automatically:

```sh
flox activate -r barstoolbluz/setup-1pass
```

This wizard will guide you through the process of creating the necessary configuration.

#### Option 2: Manual Setup

Alternatively, you can set up the 1Password CLI manually by creating the config file yourself. The file should be structured like this:

```json
{
        "latest_signin": "your_organization",
        "device": "2xdzachbrog69jockvku6qshakeyerhips",
        "accounts": [
                {
                        "shorthand": "your_org_shorthand",
                        "accountUUID": "H4D9BhyE9WmS7-MbaG0qWDsaOi",
                        "url": "https://your-team.1password.com",
                        "email": "your.email@example.com",
                        "accountKey": "Y6bXKI-Z6_9ONAkwYRbFnQRcn3lyIEY4DDpgkURh",
                        "userUUID": "GR9EqmcQVmIGa6XCIpW8hue6Ef"
                }
        ]
}
```

You'll need to replace the example values with your actual 1Password account information.

## Usage 🛠️

### Activate the Environment
```sh
flox activate
```

Once the environment is activated, you can use `git`, `gh`, and `aws` commands as you normally would. The wrapper functions will handle credential management transparently:

### GitHub CLI (gh)
```sh
# Check auth status
gh auth status

# List repositories
gh repo list

# Clone a repository
gh repo clone org/repo
```

### Git
```sh
# Push to a repository (auth handled automatically)
git push origin main

# Pull from a repository
git pull

# Clone a private repository
git clone https://github.com/organization/private-repo.git
```

### AWS CLI
```sh
# List S3 buckets
aws s3 ls

# Describe EC2 instances
aws ec2 describe-instances
```

## Compatible Systems

This environment is compatible with:
- aarch64-darwin (Apple Silicon Macs)
- aarch64-linux
- x86_64-darwin (Intel Macs)
- x86_64-linux

## Non-Interactive Shells ⚠️

The wrapper functions for `git`, `gh`, and `aws` are written to `$FLOX_ENV_CACHE/shell/`. This is necessary because:

- Wrapper functions defined interactively aren't available in non-interactive scripts
- `bash -i` doesn't always work as expected for sourcing interactive functions
- Scripts running in this Flox environment **must source** the relevant wrapper script before calling `git`, `gh`, or `aws`

For example, in a non-interactive bash script:

```sh
source "$FLOX_ENV_CACHE/shell/wrapper.bash"
git push origin main
```

For zsh scripts:

```sh
source "$FLOX_ENV_CACHE/shell/wrapper.zsh"
gh repo list
```

For fish scripts:

```sh
source "$FLOX_ENV_CACHE/shell/wrapper.fish"
aws s3 ls
```

For interactive sessions, Flox automatically sources the appropriate wrapper script depending on which shell it's running in.

## Shell Compatibility

- **Bash** – `wrapper.bash`
- **Zsh** – `wrapper.zsh`
- **Fish** – `wrapper.fish`

Each shell wrapper implements proper trap-based cleanup mechanisms specific to its environment to ensure secure handling of credentials even if commands fail or are interrupted.

## Notes

### Session Token Handling

#### Local Environment
The environment caches 1Password session tokens under:

```sh
$HOME/.config/op/1password-session.token
```

#### CI Environment
In CI environments, session tokens are kept exclusively in memory as environment variables to eliminate file I/O operations and potential points of failure.

#### About 1Password Session Tokens

A session token is a temporary authentication credential that allows the 1Password CLI to access your vault without requiring you to enter your password for every operation.

**Security considerations:**

- **Convenience vs. Security**: Caching the token provides convenience by allowing you to use 1Password-integrated commands without re-authenticating, even if you temporarily exit the Flox environment.
- **Time-limited**: Session tokens are temporary and expire after a period of inactivity (typically 30 minutes), limiting exposure.
- **Local storage only**: In local environments, the token is stored only on your local machine and is protected with file permissions (chmod 600).
- **CI security**: In CI environments, tokens remain in memory only and are never written to disk.
- **Risk awareness**: If your system is compromised while a valid session token exists, an attacker could potentially access your 1Password vault until the token expires.

This approach balances security and usability. For higher security environments, you may want to modify the environment to avoid caching the token, though this will require re-authentication more frequently.

## Extensibility

This pattern can be extended to other CLI tools that require credentials. For example, you could add similar wrappers for:
- Databricks CLI
- Snowflake CLI
- Azure CLI
- Google Cloud Platform SDK
- Terraform CLI
- OpenStack CLI

## How to Extend

To extend this pattern to additional tools, you'll need to:

1. **Define environment variables** in the `[vars]` section of your `manifest.toml` for each credential:

```toml
[vars]
# Existing vars
OP_GITHUB_VAULT = "vault"
# ...

# New vars for your tool (e.g., Databricks)
OP_DATABRICKS_VAULT = "vault"              # Vault containing Databricks credentials
OP_DATABRICKS_ITEM = "databricks_creds"    # Item storing Databricks credentials
OP_DATABRICKS_HOST_FIELD = "host"          # Field for Databricks host
OP_DATABRICKS_TOKEN_FIELD = "token"        # Field for Databricks token
```

2. **Create a wrapper function** in the appropriate section of the `on-activate` hook in your `manifest.toml`:

```bash
# For tools that use environment variables
toolname() { 
  op run --session "$OP_SESSION_TOKEN" --env-file <(echo -e "ENV_VAR1=op://$OP_TOOL_VAULT/$OP_TOOL_ITEM/$OP_TOOL_FIELD1\nENV_VAR2=op://$OP_TOOL_VAULT/$OP_TOOL_ITEM/$OP_TOOL_FIELD2") -- toolname "$@"; 
}

# Example for Databricks CLI
databricks() { 
  op run --session "$OP_SESSION_TOKEN" --env-file <(echo -e "DATABRICKS_HOST=op://$OP_DATABRICKS_VAULT/$OP_DATABRICKS_ITEM/$OP_DATABRICKS_HOST_FIELD\nDATABRICKS_TOKEN=op://$OP_DATABRICKS_VAULT/$OP_DATABRICKS_ITEM/$OP_DATABRICKS_TOKEN_FIELD") -- databricks "$@"; 
}
```

3. **Make your wrapper environment-aware** by including proper environment detection and error handling:

```bash
detect_environment() {
  # Check for GitHub Actions
  if [ -n "$GITHUB_ACTIONS" ]; then
    echo "ci"
    return
  fi
  
  # Add checks for other CI platforms as needed
  # if [ -n "$JENKINS_URL" ]; then echo "ci"; return; fi
  # if [ -n "$GITLAB_CI" ]; then echo "ci"; return; fi
  
  # Default to local environment
  echo "local"
}

toolname() {
  local env_type=$(detect_environment)
  
  # Check for existing session token
  if [ "$env_type" = "ci" ]; then
    # CI environment - use environment variables
    if [ -z "$OP_SESSION_TOKEN" ]; then
      # Try to authenticate using service account
      if [ -z "$OP_SERVICE_ACCOUNT_TOKEN" ]; then
        echo "Error: OP_SERVICE_ACCOUNT_TOKEN not set in CI environment" >&2
        return 1
      fi
      export OP_SESSION_TOKEN="$OP_SERVICE_ACCOUNT_TOKEN"
    fi
  else
    # Local environment - use file-based token with fallback to interactive login
    # (existing code)
  fi
  
  # Proceed with command execution
  op run --session "$OP_SESSION_TOKEN" --env-file <(echo -e "ENV_VAR1=op://$OP_TOOL_VAULT/$OP_TOOL_ITEM/$OP_TOOL_FIELD1\nENV_VAR2=op://$OP_TOOL_VAULT/$OP_TOOL_ITEM/$OP_TOOL_FIELD2") -- toolname "$@"
}
```

The `on-activate` hook will automatically add these functions to the shell wrapper files in `$FLOX_ENV_CACHE/shell/`, making them available for both interactive and non-interactive shells.

Note that different tools may require different numbers of credentials. Two key aspects to research:

1. **Tool requirements**: Identify which environment variables each tool expects for authentication:
   - Single token (like GitHub)
   - Key/secret pair (like AWS)
   - Host/token combination (like Databricks)
   - Multiple credentials (some cloud providers)

2. **1Password structure**: Determine how these credentials are stored in your 1Password vault:
   - Which vault contains the credentials
   - The name of the item storing the credentials
   - The exact field names used within that item

You'll need to match your 1Password structure to the environment variables required by each tool. This might involve creating new items or fields in your 1Password vault specifically organized for use with this environment.

For tools that don't accept environment variables, you'll need a custom approach similar to the Git wrapper, using techniques appropriate for that specific tool's authentication mechanism.
