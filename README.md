# github-actions-template
GitHub Actions Template
# 🚀 My Composite GitHub Action

This action prints a message and uses a GitHub token. It's intended as a reusable component in GitHub Actions workflows.

## 📥 Inputs

| Name    | Description       | Required | Default               |
|---------|-------------------|----------|------------------------|
| `token` | GitHub token      | ✅       | —                      |
| `message` | Message to print | ❌       | `"Hello from the Action!"` |

## 📤 Outputs

## 📦 Usage

```yaml
jobs:
  example:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: your-org/gh-action-template@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          message: "Hi from workflow"

## 🖼️ Examples

