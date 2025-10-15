# login-aws

GitHub Action for authenticating to AWS using OIDC (OpenID Connect) or access keys, with optional ECR login and EKS configuration.

## Features

- 🔐 **Secure OIDC Authentication** - Recommended passwordless auth
- 🔑 **Access Key Support** - Fallback for environments without OIDC
- 🐳 **ECR Integration** - Optional Docker registry login
- ☸️ **EKS Support** - Configure kubectl for your clusters
- 📦 **CodeArtifact Support** - Configure package managers (npm, pip, etc.)
- 🔄 **Cross-Account Access** - Support for multiple ECR registries
- ✅ **Automatic Verification** - Confirms successful authentication

## Quick Start

```yaml
name: Deploy to AWS
on: [push]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # Required for OIDC
      contents: read
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Login to AWS
        uses: KoalaOps/login-aws@v1
        with:
          role_to_assume: arn:aws:iam::123456789012:role/github-actions
          aws_region: us-east-1
```

## Prerequisites

### Configure OIDC in AWS

1. Create an OIDC identity provider in AWS IAM:
   - Provider URL: `https://token.actions.githubusercontent.com`
   - Audience: `sts.amazonaws.com`

2. Create an IAM role with a trust policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YourOrg/YourRepo:*"
        }
      }
    }
  ]
}
```

## Authentication Methods

### OIDC (Recommended)
Use `role_to_assume` for passwordless authentication via GitHub's OIDC token.

### Access Keys (Alternative)
Use `aws_access_key_id` and `aws_secret_access_key` for traditional authentication. Only use when OIDC is not available.

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `aws_region` | AWS region to use | ✅ | - |
| **OIDC Authentication** |
| `role_to_assume` | ARN of IAM role to assume via OIDC | ⚠️* | - |
| `role_session_name` | Name for the assumed role session | ❌ | `github-actions-{run-id}` |
| `role_duration` | Session duration in seconds | ❌ | `3600` |
| **Access Key Authentication** |
| `aws_access_key_id` | AWS Access Key ID | ⚠️* | - |
| `aws_secret_access_key` | AWS Secret Access Key | ⚠️* | - |
| **Services** |
| `enable_ecr_login` | Enable Docker login to ECR | ❌ | `false` |
| `eks_cluster_name` | EKS cluster to configure kubectl for | ❌ | - |
| `ecr_registries` | AWS account IDs for cross-account ECR access (comma-separated) | ❌ | - |
| `ecr_repositories` | ECR repository to create if missing (currently limited to single repository) | ❌ | - |
| `codeartifact_domain` | CodeArtifact domain name | ❌ | - |
| `codeartifact_repository` | CodeArtifact repository name | ❌ | - |
| `codeartifact_tool` | Tool to configure (npm, pip, twine, dotnet, nuget, swift, maven, gradle) | ❌ | - |
| `codeartifact_region` | CodeArtifact region (defaults to `aws_region`) | ❌ | - |
| `codeartifact_domain_owner` | AWS account ID that owns the domain (defaults to authenticated account) | ❌ | - |
| `codeartifact_duration` | Token duration in seconds | ❌ | `43200` (12 hours) |

*Either use OIDC (`role_to_assume`) or access keys (`aws_access_key_id` + `aws_secret_access_key`)

## Quick Task Reference

| Task | Required Inputs | Optional Inputs |
|------|-----------------|------------------|
| **EKS only** | `aws_region`, `eks_cluster_name`, auth* | - |
| **ECR only** | `aws_region`, `enable_ecr_login: true`, auth* | `ecr_repositories`, `ecr_registries` |
| **CodeArtifact only** | `aws_region`, `codeartifact_domain`, `codeartifact_repository`, `codeartifact_tool`, auth* | `codeartifact_region`, `codeartifact_domain_owner`, `codeartifact_duration` |
| **Both EKS & ECR** | `aws_region`, `eks_cluster_name`, `enable_ecr_login: true`, auth* | `ecr_repositories`, `ecr_registries` |
| **Cross-account ECR** | `aws_region`, `enable_ecr_login: true`, `ecr_registries`, auth* | - |

*auth = either `role_to_assume` (OIDC) or `aws_access_key_id` + `aws_secret_access_key`

## Outputs

| Output | Description |
|--------|-------------|
| `aws_account_id` | AWS Account ID |
| `ecr_registry` | ECR registry URL (if ECR login enabled) |
| `ecr_logged_in` | `true` if ECR login was performed, `false` otherwise |
| `kubectl_context` | Kubernetes context name (if EKS configured) |
| `codeartifact_logged_in` | `true` if CodeArtifact login was performed, `false` otherwise |
| `codeartifact_token` | CodeArtifact authorization token (only set for maven/gradle) |
| `codeartifact_endpoint` | CodeArtifact repository endpoint URL (only set for maven/gradle) |

## ECR Parameters Explained

### When to use `ecr_registries`
Use when you need to access ECR registries in **other AWS accounts**:
- Pulling base images from a shared account
- Pushing to a central registry account
- Multi-account deployment pipelines

**Format:** Comma-separated AWS account IDs
**Example:** `ecr_registries: "123456789012,987654321098"`

### When to use `ecr_repositories`
Use when you want to **ensure a repository exists** in your account:
- Auto-creating a repo for a new service
- CI/CD pipelines that may run before infrastructure setup
- Avoiding "repository not found" errors

Example: `ecr_repositories: "my-app"`

**Note:** Currently limited to a single repository. For multiple repositories, run the action multiple times or create them separately.

**Note:** These can be used together - `ecr_registries` for cross-account access, `ecr_repositories` for repo creation.

## CodeArtifact Parameters Explained

### Supported Tools
CodeArtifact supports multiple package managers:
- **npm** - Node.js package manager
- **pip** - Python package installer
- **twine** - Python package uploader
- **dotnet** - .NET package manager
- **nuget** - NuGet package manager
- **swift** - Swift package manager
- **maven** - Java/JVM package manager (token-based auth)
- **gradle** - Java/JVM build tool (token-based auth)

**Note:** For Maven and Gradle, this action exports environment variables (`CODEARTIFACT_AUTH_TOKEN` and `CODEARTIFACT_REPO_URL`) that you'll reference in your configuration files, since the AWS CLI `login` command doesn't support these tools natively.

### Required Parameters
To enable CodeArtifact login, you must provide:
- `codeartifact_domain` - Your CodeArtifact domain name
- `codeartifact_repository` - The repository within the domain
- `codeartifact_tool` - Which package manager to configure

### Optional Parameters
- `codeartifact_region` - Defaults to the main `aws_region` input
- `codeartifact_domain_owner` - Defaults to the authenticated AWS account
- `codeartifact_duration` - Token lifetime in seconds (default: 43200 = 12 hours)

### Cross-Account Access
For cross-account CodeArtifact access, specify the `codeartifact_domain_owner` parameter with the AWS account ID that owns the domain.

## Examples

### With Access Keys (Non-OIDC)

```yaml
- name: Login to AWS with access keys
  uses: KoalaOps/login-aws@v1
  with:
    aws_access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws_secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws_region: us-east-1
    enable_ecr_login: true
```

### With ECR for Docker Operations

```yaml
- name: Login to AWS with ECR
  uses: KoalaOps/login-aws@v1
  with:
    role_to_assume: ${{ secrets.AWS_ROLE_ARN }}
    aws_region: us-east-1
    enable_ecr_login: true
    ecr_repositories: my-app  # Optional: ensures repository exists

- name: Build and push Docker image
  run: |
    docker build -t ${{ steps.login-aws.outputs.ecr_registry }}/my-app:latest .
    docker push ${{ steps.login-aws.outputs.ecr_registry }}/my-app:latest
```

### Ensure ECR Repository Exists

```yaml
- name: Login and setup ECR repo
  uses: KoalaOps/login-aws@v1
  with:
    role_to_assume: ${{ secrets.AWS_ROLE_ARN }}
    aws_region: us-east-1
    enable_ecr_login: true
    ecr_repositories: backend  # Currently limited to single repository
```

### With EKS for Kubernetes Operations

```yaml
- name: Login to AWS with EKS
  uses: KoalaOps/login-aws@v1
  with:
    role_to_assume: ${{ secrets.AWS_ROLE_ARN }}
    aws_region: us-east-1
    eks_cluster_name: production-cluster

- name: Deploy to Kubernetes
  run: |
    kubectl apply -f k8s/deployment.yaml
    kubectl rollout status deployment/my-app
```

### Cross-Account ECR Access

```yaml
- name: Login to AWS with multiple ECR registries
  id: aws
  uses: KoalaOps/login-aws@v1
  with:
    role_to_assume: ${{ secrets.AWS_ROLE_ARN }}
    aws_region: us-east-1
    enable_ecr_login: true
    ecr_registries: "123456789012,987654321098"  # AWS Account IDs

- name: Push to multiple registries
  run: |
    # Build image
    docker build -t my-app:latest .

    # Push to primary account
    docker tag my-app:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
    docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest

    # Push to secondary account
    docker tag my-app:latest 987654321098.dkr.ecr.us-east-1.amazonaws.com/shared/my-app:latest
    docker push 987654321098.dkr.ecr.us-east-1.amazonaws.com/shared/my-app:latest
```

### CodeArtifact for NPM

```yaml
- name: Login to AWS with CodeArtifact
  uses: KoalaOps/login-aws@v1
  with:
    role_to_assume: ${{ secrets.AWS_ROLE_ARN }}
    aws_region: us-east-1
    codeartifact_domain: my-artifacts
    codeartifact_repository: npm-store
    codeartifact_tool: npm

- name: Install dependencies
  run: npm install

- name: Publish package
  run: npm publish
```

### CodeArtifact for Python/Pip

```yaml
- name: Login to AWS with CodeArtifact
  uses: KoalaOps/login-aws@v1
  with:
    role_to_assume: ${{ secrets.AWS_ROLE_ARN }}
    aws_region: us-east-1
    codeartifact_domain: my-artifacts
    codeartifact_repository: pypi-store
    codeartifact_tool: pip

- name: Install dependencies
  run: pip install -r requirements.txt

- name: Build and publish
  run: |
    python -m build
    twine upload dist/*
```

### Cross-Account CodeArtifact

```yaml
- name: Login to AWS with cross-account CodeArtifact
  uses: KoalaOps/login-aws@v1
  with:
    role_to_assume: ${{ secrets.AWS_ROLE_ARN }}
    aws_region: us-east-1
    codeartifact_domain: shared-artifacts
    codeartifact_domain_owner: "123456789012"  # Different account
    codeartifact_repository: npm-shared
    codeartifact_tool: npm
    codeartifact_duration: 43200  # 12 hours
```

### CodeArtifact for Maven

```yaml
- name: Login to AWS with CodeArtifact
  uses: KoalaOps/login-aws@v1
  with:
    role_to_assume: ${{ secrets.AWS_ROLE_ARN }}
    aws_region: us-east-1
    codeartifact_domain: my-artifacts
    codeartifact_repository: maven-repo
    codeartifact_tool: maven

- name: Build with Maven
  run: mvn clean install

- name: Publish to CodeArtifact
  run: mvn deploy
```

**Required `~/.m2/settings.xml` configuration:**

```xml
<settings>
  <servers>
    <server>
      <id>codeartifact</id>
      <username>aws</username>
      <password>${env.CODEARTIFACT_AUTH_TOKEN}</password>
    </server>
  </servers>

  <profiles>
    <profile>
      <id>codeartifact</id>
      <repositories>
        <repository>
          <id>codeartifact</id>
          <url>${env.CODEARTIFACT_REPO_URL}</url>
        </repository>
      </repositories>
    </profile>
  </profiles>

  <activeProfiles>
    <activeProfile>codeartifact</activeProfile>
  </activeProfiles>
</settings>
```

Alternatively, you can create the settings.xml in your workflow:

```yaml
- name: Login to AWS with CodeArtifact
  uses: KoalaOps/login-aws@v1
  with:
    role_to_assume: ${{ secrets.AWS_ROLE_ARN }}
    aws_region: us-east-1
    codeartifact_domain: my-artifacts
    codeartifact_repository: maven-repo
    codeartifact_tool: maven

- name: Create Maven settings.xml
  run: |
    mkdir -p ~/.m2
    cat > ~/.m2/settings.xml << 'EOF'
    <settings>
      <servers>
        <server>
          <id>codeartifact</id>
          <username>aws</username>
          <password>${env.CODEARTIFACT_AUTH_TOKEN}</password>
        </server>
      </servers>
      <profiles>
        <profile>
          <id>codeartifact</id>
          <repositories>
            <repository>
              <id>codeartifact</id>
              <url>${env.CODEARTIFACT_REPO_URL}</url>
            </repository>
          </repositories>
        </profile>
      </profiles>
      <activeProfiles>
        <activeProfile>codeartifact</activeProfile>
      </activeProfiles>
    </settings>
    EOF

- name: Build and deploy
  run: mvn clean deploy
```

### CodeArtifact for Gradle

```yaml
- name: Login to AWS with CodeArtifact
  uses: KoalaOps/login-aws@v1
  with:
    role_to_assume: ${{ secrets.AWS_ROLE_ARN }}
    aws_region: us-east-1
    codeartifact_domain: my-artifacts
    codeartifact_repository: maven-repo
    codeartifact_tool: gradle

- name: Build with Gradle
  run: ./gradlew build

- name: Publish to CodeArtifact
  run: ./gradlew publish
```

**Required `build.gradle` configuration:**

```groovy
repositories {
    maven {
        url System.env.CODEARTIFACT_REPO_URL
        credentials {
            username "aws"
            password System.env.CODEARTIFACT_AUTH_TOKEN
        }
    }
}

publishing {
    repositories {
        maven {
            url System.env.CODEARTIFACT_REPO_URL
            credentials {
                username "aws"
                password System.env.CODEARTIFACT_AUTH_TOKEN
            }
        }
    }
}
```

**Or for Kotlin DSL (`build.gradle.kts`):**

```kotlin
repositories {
    maven {
        url = uri(System.getenv("CODEARTIFACT_REPO_URL") ?: "")
        credentials {
            username = "aws"
            password = System.getenv("CODEARTIFACT_AUTH_TOKEN") ?: ""
        }
    }
}

publishing {
    repositories {
        maven {
            url = uri(System.getenv("CODEARTIFACT_REPO_URL") ?: "")
            credentials {
                username = "aws"
                password = System.getenv("CODEARTIFACT_AUTH_TOKEN") ?: ""
            }
        }
    }
}
```

### Long-Running Jobs

```yaml
- name: Login to AWS for long operation
  uses: KoalaOps/login-aws@v1
  with:
    role_to_assume: ${{ secrets.AWS_ROLE_ARN }}
    aws_region: us-east-1
    role_duration: 7200  # 2 hours
```

## Building Blocks Used

This action internally uses:
- `aws-actions/configure-aws-credentials@v4` - AWS OIDC authentication
- `aws-actions/amazon-ecr-login@v2` - ECR Docker login
- `KoalaOps/ensure-ecr-repository@v1` - ECR repository creation (when ecr_repositories provided)

## Security Best Practices

1. **Use OIDC** - Never store AWS access keys as secrets
2. **Least Privilege** - Grant minimal required permissions to the IAM role
3. **Restrict Trust Policy** - Limit which repositories and branches can assume the role
4. **Short Sessions** - Use the minimum `role_duration` needed

## Troubleshooting

### Error: "Could not assume role"

Ensure your repository has the `id-token: write` permission:

```yaml
permissions:
  id-token: write
  contents: read
```

### Error: "Not authorized to perform sts:AssumeRoleWithWebIdentity"

Check your IAM role's trust policy includes the correct repository pattern:

```json
"StringLike": {
  "token.actions.githubusercontent.com:sub": "repo:YourOrg/YourRepo:*"
}
```

### ECR Login Fails

Ensure the IAM role has the necessary ECR permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "arn:aws:ecr:*:*:repository/*"
    }
  ]
}
```

For repository creation (`ecr_repositories`), add:
```json
{
  "Effect": "Allow",
  "Action": [
    "ecr:CreateRepository",
    "ecr:DescribeRepositories"
  ],
  "Resource": "*"
}
```

### EKS Access Fails

Ensure the IAM role has the necessary EKS permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:ListClusters"
      ],
      "Resource": "*"
    }
  ]
}
```

**Note:** You also need to be mapped in the EKS cluster's aws-auth ConfigMap or use EKS access entries.

### CodeArtifact Login Fails

Ensure the IAM role has the necessary CodeArtifact permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "codeartifact:GetAuthorizationToken",
        "codeartifact:GetRepositoryEndpoint",
        "codeartifact:ReadFromRepository"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "sts:GetServiceBearerToken",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "sts:AWSServiceName": "codeartifact.amazonaws.com"
        }
      }
    }
  ]
}
```

For publishing packages, add:
```json
{
  "Effect": "Allow",
  "Action": [
    "codeartifact:PublishPackageVersion",
    "codeartifact:PutPackageMetadata"
  ],
  "Resource": "*"
}
```

## Support

- 📖 [Documentation](https://github.com/KoalaOps/login-aws-action)
- 🐛 [Issue Tracker](https://github.com/KoalaOps/login-aws-action/issues)
- 💬 [Discussions](https://github.com/KoalaOps/login-aws-action/discussions)

## License

MIT
