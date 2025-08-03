<div style="display: flex; align-items: center;">
  <img src="logo.png" alt="DevOpspolis Logo" width="50" height="50" style="margin-right: 10px;"/>
  <span style="font-size: 2.2em;">Deploy to AWS Lambda</span>
</div>

![GitHub Marketplace](https://img.shields.io/badge/GitHub%20Marketplace-Deploy%20to%20AWS%20Lambda-blue?logo=github)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

<p>
GitHub Action to update an existing AWS Lambda from a ZIP file, GitHub artifact, or container image, and optionally apply configuration and environment settings from multiple sources including files, AWS Secrets Manager, and AWS Systems Manager Parameter Store.
</p>

See more [GitHub Actions by DevOpspolis](https://github.com/marketplace?query=devopspolis&type=actions)

---

## 📚 Table of Contents
- [✨ Features](#features)
- [📥 Inputs](#inputs)
- [📤 Outputs](#outputs)
- [📦 Usage](#usage)
- [🚦 Requirements](#requirements)
- [🧑‍⚖️ Legal](#legal)

---
<!-- trunk-ignore(markdownlint/MD033) -->
<a id="features"></a>
## ✨ Features
- **Flexible Code Deployment**: Updates function code from S3 bucket, ECR container registry, or GitHub artifact
- **Multi-Source Configuration**: Load configuration from files, AWS Secrets Manager, or AWS Systems Manager Parameter Store
- **Multi-Source Environment Variables**: Load environment variables from multiple sources with conflict resolution
- **Layer Management**: Updates Lambda layers with ARN resolution
- **Version Publishing**: Publishes new Lambda versions with automatic VERSION environment variable updates
- **Alias Management**: Creates and updates Lambda aliases
- **IAM Role Assumption**: Supports cross-account deployments
- **Comprehensive Error Handling**: Detailed error messages and validation
- **Automatic Cleanup**: Cleans up temporary files and configurations

---
<!-- trunk-ignore(markdownlint/MD033) -->
<a id="inputs"></a>
## 📥 Inputs

| Name                   | Description                                                            | Required | Default |
|------------------------|------------------------------------------------------------------------|----------|---------|
| `function_name`        | Name of the Lambda function to deploy                                  | true     | —       |
| `version`              | Publishes a new Lambda version. The value provided can be used to update the VERSION environment variable. See `environment_variables` input below | false | — |
| `aliases`              | Comma-separated list of aliases to update                              | false    | —       |
| `uri`                  | S3 URI (`s3://bucket/key.zip`) or ECR URI (`repository:tag`) to deploy. Either *uri* or *artifact* are required | false | —       |
| `artifact`             | GitHub Actions artifact name and ZIP filename (artifact:filename). Either *artifact* or *uri* are required. If both are provided then *uri* takes precedence | false | — |
| `configuration`        | Space-separated list of configuration sources. Each source can be: `file::path/to/file.json`, `secretsmanager::secret-name`, or `ssm::parameter-name`. Later sources override earlier ones for conflicting keys. | false | — |
| `environment_variables` | Space-separated list of environment variable sources. Each source can be: `file::path/to/file.json`, `secretsmanager::secret-name`, or `ssm::parameter-name`. Later sources override earlier ones for conflicting keys. If `VERSION` variable is declared and input `version` was provided, the Lambda environment variable `VERSION` will be set to the `version` value. | false | — |
| `layers`               | Comma-separated list of Lambda layer ARNs or names                     | false    | —       |
| `role`                 | IAM role ARN or name to assume for deployment                          | false    | —       |
| `aws_region`           | AWS region for deployment                                               | false    | us-east-1 |

### Configuration and Environment Variable Sources

The `configuration` and `environment_variables` inputs support multiple source types:

- **File**: `file::path/to/config.json` - Load from local file
- **Secrets Manager**: `secretsmanager::my-secret` - Load from AWS Secrets Manager
- **Parameter Store**: `ssm::my-parameter` - Load from AWS Systems Manager Parameter Store
- **Legacy**: `my-secret` - Defaults to Secrets Manager for backward compatibility

**Conflict Resolution**: When multiple sources contain the same key, later sources override earlier ones (last-wins strategy).

---
<!-- trunk-ignore(markdownlint/MD033) -->
<a id="outputs"></a>
## 📤 Outputs

| Name              | Description                                        |
|-------------------|----------------------------------------------------|
| `function_name`   | The function name                                  |
| `function_version`| The version or blank if not specified              |

---
<!-- trunk-ignore(markdownlint/MD033) -->
<a id="usage"></a>
## 📦 Usage

#### Example 1 – Deploy from S3 ZIP with multiple configuration sources

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy Lambda
        uses: devopspolis/deploy-to-aws-lambda@main
        with:
          function_name: my-lambda
          version: v1.2.0
          uri: s3://my-deployment-bucket/my-lambda.zip
          configuration: >
            file::./config/base.json
            secretsmanager::lambda/my-lambda/config
            ssm::/lambda/my-lambda/overrides
          environment_variables: >
            secretsmanager::lambda/shared/env
            secretsmanager::lambda/my-lambda/env
```

#### Example 2 – Deploy GitHub artifact with layers and aliases

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy Lambda artifact
        uses: devopspolis/deploy-to-aws-lambda@main
        with:
          function_name: my-lambda-func
          artifact: lambda-build:dist/lambda.zip
          version: v1.2.0
          aliases: prod,latest
          layers: |
            arn:aws:lambda:us-west-2:123456789012:layer:shared-utils:3,
            logger-layer
          environment_variables: >
            file::./env/base.json
            secretsmanager::lambda/my-lambda/env
```

#### Example 3 – Deploy container image with role assumption

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy Lambda container
        uses: devopspolis/deploy-to-aws-lambda@main
        with:
          function_name: my-container-lambda
          uri: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-func:latest
          role: arn:aws:iam::123456789012:role/github-lambda-deploy
          aws_region: us-east-1
```

#### Example 4 – Multi-environment deployment with Parameter Store

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [dev, staging, prod]
    steps:
      - name: Deploy Lambda
        uses: devopspolis/deploy-to-aws-lambda@main
        with:
          function_name: my-lambda-${{ matrix.environment }}
          artifact: lambda-build:lambda.zip
          version: ${{ github.sha }}
          configuration: >
            file::./config/base.json
            ssm::/lambda/shared/config
            ssm::/lambda/${{ matrix.environment }}/config
          environment_variables: >
            secretsmanager::lambda/shared/env
            ssm::/lambda/${{ matrix.environment }}/env
```

### Configuration Examples

#### Base Configuration File (`./config/base.json`)
```json
{
  "Description": "My Lambda function",
  "MemorySize": 512,
  "Timeout": 30,
  "Runtime": "python3.9"
}
```

#### AWS Secrets Manager Secret (`lambda/my-lambda/config`)
```json
{
  "MemorySize": 1024,
  "Timeout": 300,
  "Role": "my-lambda-execution-role",
  "VpcConfig": {
    "SubnetIds": ["subnet-12345", "subnet-67890"],
    "SecurityGroupIds": ["sg-abcdef"]
  }
}
```

#### Environment Variables from Multiple Sources

**Shared Environment (`lambda/shared/env`)**:
```json
{
  "LOG_LEVEL": "INFO",
  "REGION": "us-east-1"
}
```

**Function-specific Environment (`lambda/my-lambda/env`)**:
```json
{
  "DATABASE_URL": "postgres://...",
  "API_KEY": "secret-key",
  "VERSION": "placeholder"
}
```

**Result**: The `VERSION` placeholder will be replaced with the actual version if the `version` input is provided.

<!-- trunk-ignore(markdownlint/MD033) -->
<a id="requirements"></a>
## 🚦 Requirements

### AWS Permissions
Your GitHub workflow must provide AWS credentials with the following permissions:

#### Lambda Permissions
- `lambda:UpdateFunctionCode`
- `lambda:UpdateFunctionConfiguration`
- `lambda:PublishVersion`
- `lambda:UpdateAlias`
- `lambda:CreateAlias`
- `lambda:ListAliases`
- `lambda:GetFunction` (for wait operations)

#### Secrets Manager Permissions (if using `secretsmanager::` sources)
- `secretsmanager:GetSecretValue`
- `secretsmanager:DescribeSecret`

#### Systems Manager Permissions (if using `ssm::` sources)
- `ssm:GetParameter`
- `ssm:GetParameters`

#### IAM Permissions (if using `role` input)
- `sts:AssumeRole`
- `sts:GetCallerIdentity`

#### S3 Permissions (if deploying from S3)
- `s3:GetObject`

### Example IAM Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "lambda:UpdateFunctionCode",
        "lambda:UpdateFunctionConfiguration",
        "lambda:PublishVersion",
        "lambda:UpdateAlias",
        "lambda:CreateAlias",
        "lambda:ListAliases",
        "lambda:GetFunction"
      ],
      "Resource": "arn:aws:lambda:*:*:function:my-lambda*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:*:*:secret:lambda/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters"
      ],
      "Resource": "arn:aws:ssm:*:*:parameter/lambda/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sts:GetCallerIdentity"
      ],
      "Resource": "*"
    }
  ]
}
```

## 🔧 Advanced Features

### Conflict Resolution
When multiple sources contain the same configuration key or environment variable:
- Sources are processed in the order specified
- Later sources override earlier ones (last-wins)
- This allows for layered configuration (base → environment-specific → function-specific)

### Automatic Resource Resolution
- **Short IAM role names**: Automatically resolved to full ARNs using account ID
- **Short layer names**: Automatically resolved to full ARNs using account ID and region
- **VERSION environment variable**: Automatically updated when `version` input is provided

### Error Handling
- Comprehensive validation of all inputs
- Detailed error messages with actionable guidance
- Proper cleanup of temporary files even on failure
- Graceful handling of missing resources

### Backward Compatibility
- Sources without prefixes default to Secrets Manager
- Maintains compatibility with existing workflows
- Existing input parameter names unchanged

<!-- trunk-ignore(markdownlint/MD033) -->
<a id="legal"></a>
## 🧑‍⚖️ Legal
The MIT License (MIT)