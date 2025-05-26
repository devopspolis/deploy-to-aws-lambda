<div style="display: flex; align-items: center;">
  <img src="logo.png" alt="DevOpspolis Logo" width="50" height="50" style="margin-right: 10px;"/>
  <span style="font-size: 2.2em;">Deploy to AWS Lambda</span>
</div>

![GitHub Marketplace](https://img.shields.io/badge/GitHub%20Marketplace-Deploy%20to%20AWS%20Lambda-blue?logo=github)
![License](https://img.shields.io/github/license/devopspolis/deploy-to-aws-lambda)

<p>
GitHub Action to deploy an existing AWS Lambda from a ZIP file, GitHub artifact, or container image, and optionally apply configuration and environment settings from AWS Secrets Manager.
</p>




---

## 📚 Table of Contents
- [✨ Features](#features)
- [📥 Inputs](#inputs)
- [📤 Outputs](#outputs)
- [📦 Usage](#usage)
- [🚦 Requirements](#requirements)

---
<!-- trunk-ignore(markdownlint/MD033) -->
<a id="features"></a>
## ✨ Features
- Updates function code from either S3 bucket, ECR container registry, or GitHub artifact
- Updates lambda layers
- Version publishing
- Updates configuration
- Updates environment variables, including auto-update of VERSION variable
- Updates aliases
- Auto-Rollback on failure

---
<!-- trunk-ignore(markdownlint/MD033) -->
<a id="inputs"></a>
## 📥 Inputs

| Name                   | Description                                                            | Required | Default |
|------------------------|------------------------------------------------------------------------|----------|---------|
| `function_name`        | Name of the Lambda function to deploy                                  | true     | —       |
| `version`              | Publish a new Lambda version                                           | false    | —       |
| `aliases`              | Comma-separated list of aliases to update                              | false    | —       |
| `uri`                  | S3 URI (`s3://bucket/key.zip`) or ECR URI (`repository:tag`) to deploy. Either *uri* or *artifact* are required | false | —       |
| `artifact`             | GitHub Actions artifact name and ZIP filename (artifact:filename). Either *artifact* or *uri* are required. If both are provided then *uri* takes precedence             | false    | —       |
| `configuration_secret` | AWS Secrets Manager secret with Lambda config (key-value pairs)        | false    | —       |
| `environment_secret`   | AWS Secrets Manager secret with env vars (key-value pairs). If `VERSION` variable is declared and input `version` was provided then the Lambda environment variable `VERSION` will be set to the `version` value         | false    | —       |
| `layers`               | Comma-separated list of Lambda layer ARNs                              | false    | —       |
| `role-to-assume`       | IAM role ARN or name to assume for deployment                          | false    | —       |

---
<!-- trunk-ignore(markdownlint/MD033) -->
<a id="outputs"></a>
## 📤 Outputs

| Name              | Description                                        |
|-------------------|----------------------------------------------------|
| `function_name`   | The function name                                  |
| `function_version`| The version or blank if not specified              |
| `rollback_status` | `'ok'` if successful, `'rolled_back'` if failed    |

---
<!-- trunk-ignore(markdownlint/MD033) -->
<a id="usage"></a>
## 📦 Usage

#### Example 1 – Deploy from S3 ZIP, apply configuration/environment variables from AWS Secrets Manager, and publish new version

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
          configuration_secret: lambda/my-lambda/configuration
          environment_secret: lambda/my-lambda/env/variables
```
#### lambda/my-lambda/configuration AWS Secrets Manager secret

| Secret key            | Secret value      |
|-----------------------|-------------------|
| description           | 'My great lambda' |
| memory-size           | 5308              |
| ephemeral-storage     | Size=5000         |
| timeout               | 600               |
| role                  | my-lambda-role    |

#### lambda/my-lambda/env/variables AWS Secrets Manager secret

| Secret key  | Secret value          |
|-------------|-----------------------|
| ENVIRONMENT | DEV                   |
| S3_BUCKET   | my-lambda-dev-abcd123 |
| VERSION     |                       |

#### Example 2 – Deploy GitHub artifact, publish version, update aliases, and use layers
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
```

#### Example 3 – Deploy container image and assume IAM role
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
```

<!-- trunk-ignore(markdownlint/MD033) -->
<a id="requirements"></a>
🚦 Requirements
GitHub workflow must provide AWS credentials with permission to:
- `lambda:UpdateFunctionCode`
- `lambda:UpdateFunctionConfiguration`
- `lambda:PublishVersion`
- `lambda:UpdateAlias`, `lambda:CreateAlias`
- `secretsmanager:GetSecretValue` (if using secrets)
- `sts:AssumeRole` (if `role` is used)
