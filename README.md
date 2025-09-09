# login-aws

GitHub Action for authenticating to AWS using OIDC (OpenID Connect) with optional ECR login and EKS configuration.

## Features

- 🔐 **Secure OIDC Authentication** - No long-lived credentials needed
- 🐳 **ECR Integration** - Optional Docker registry login
- ☸️ **EKS Support** - Configure kubectl for your clusters
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

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `aws_region` | AWS region to use | ✅ | - |
| `role_to_assume` | ARN of IAM role to assume via OIDC | ✅ | - |
| `role_session_name` | Name for the assumed role session | ❌ | `github-actions-{run-id}` |
| `role_duration` | Session duration in seconds | ❌ | `3600` |
| `enable_ecr_login` | Enable Docker login to ECR | ❌ | `false` |
| `eks_cluster_name` | EKS cluster to configure kubectl for | ❌ | - |
| `ecr_registries` | Comma-separated ECR registry IDs for cross-account | ❌ | - |
| `ecr_repositories` | ECR repositories to ensure exist (uses ensure-ecr-repository action) | ❌ | - |

## Outputs

| Output | Description |
|--------|-------------|
| `aws_account_id` | AWS Account ID |
| `ecr_registry` | ECR registry URL (if ECR login enabled) |
| `eks_context` | Kubernetes context name (if EKS configured) |

## Examples

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

### Ensure Multiple ECR Repositories

```yaml
- name: Login and setup multiple ECR repos
  uses: KoalaOps/login-aws@v1
  with:
    role_to_assume: ${{ secrets.AWS_ROLE_ARN }}
    aws_region: us-east-1
    enable_ecr_login: true
    ecr_repositories: backend,frontend,worker  # Uses ensure-ecr-repository action internally
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
  uses: KoalaOps/login-aws@v1
  with:
    role_to_assume: ${{ secrets.AWS_ROLE_ARN }}
    aws_region: us-east-1
    enable_ecr_login: true
    ecr_registries: "123456789012,987654321098"
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
    }
  ]
}
```

## Support

- 📖 [Documentation](https://github.com/KoalaOps/login-aws-action)
- 🐛 [Issue Tracker](https://github.com/KoalaOps/login-aws-action/issues)
- 💬 [Discussions](https://github.com/KoalaOps/login-aws-action/discussions)

## License

MIT
