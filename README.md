# Hostim GitHub Action

Restart or rebuild Hostim apps from GitHub Actions.

## Usage

```yaml
- uses: hostimdev/action@v1
  with:
    api_token: ${{ secrets.HOSTIM_API_TOKEN }}
    project: my-project-id
    app: my-app
    action: rebuild
```

## Actions

- `restart`
- `rebuild`

## Setup

1. Create an API token in Hostim
2. Add it as `HOSTIM_API_TOKEN` secret in GitHub
