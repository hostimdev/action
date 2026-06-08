# Hostim GitHub Action

Restart, rebuild, or deploy a new image to a Hostim app from GitHub Actions — and wait until the new version is live so failed deployments fail your workflow.

## Usage

### Rebuild and wait for the result

```yaml
- uses: hostimdev/action@v2
  with:
    api_token: ${{ secrets.HOSTIM_API_TOKEN }}
    project: my-project-id
    app: my-app
    action: rebuild
```

By default the action waits until the new version is **live and running**, and **fails the job** if the build fails or the rollout doesn't come up — printing the relevant warning events so you can see why.

### Deploy a specific image tag (docker apps)

```yaml
- uses: hostimdev/action@v2
  with:
    api_token: ${{ secrets.HOSTIM_API_TOKEN }}
    project: my-project-id
    app: my-app
    image: myorg/my-app:${{ github.sha }}
```

Updates the app's docker image and redeploys. `action` is not needed here.

### Restart (no build)

```yaml
- uses: hostimdev/action@v2
  with:
    api_token: ${{ secrets.HOSTIM_API_TOKEN }}
    project: my-project-id
    app: my-app
    action: restart
```

### Alert when a deploy fails

Because the action exits non-zero when a deploy fails, you can wire any failure step:

```yaml
- uses: hostimdev/action@v2
  id: deploy
  with:
    api_token: ${{ secrets.HOSTIM_API_TOKEN }}
    project: my-project-id
    app: my-app
    action: rebuild

- name: Notify client on failure
  if: failure()
  run: ./notify-client.sh "Deployment of my-app failed"
```

## Inputs

| Input            | Required | Default                    | Description |
|------------------|----------|----------------------------|-------------|
| `api_token`      | yes      | —                          | Hostim API token. |
| `project`        | yes      | —                          | Project ID. |
| `app`            | yes      | —                          | App name. |
| `action`         | no       | —                          | `restart` or `rebuild`. Optional when `image` is set. |
| `image`          | no       | —                          | New docker image incl. tag (e.g. `myorg/app:v1.2.3`). Docker apps only. Updates the image and redeploys; `action` is ignored. |
| `wait`           | no       | `true`                     | Wait until the new version is live (build, if any, plus rollout) and fail the job if it doesn't come up. Applies only when something is deployed (`rebuild` or `image`). |
| `timeout`        | no       | `600`                      | Max seconds to wait for a terminal deploy status. |

You must provide either `action` or `image`.

## How the wait works

The action waits until the new version is actually **live**, not just built. A successful
build doesn't mean a healthy deploy — the new image still has to roll out and start — so the
wait covers both, adapting to the app's deployment source:

- **Git apps** go through two phases. First the **build**: poll `buildStatus` until `failed`
  (fail) or `succeeded`. A successful build then advances to the **rollout** phase below.
- **Docker apps** don't build — they pull an image — so they start at the **rollout** phase
  directly.
- **Rollout phase** (both): poll `runtimeStatus` until `running` (pass) or `imagePullBackoff`
  (fail — usually a bad image or tag). `crashing`/`pending` are treated as transient and waited
  out (a deploy that never becomes `running` fails via the timeout).

To avoid reading the *previous* deploy's result, each phase only trusts a terminal status once
the new work has started — the build via `lastBuiltAt` (or an observed `running` build), and the
rollout via `lastDeployedAt` advancing past the baseline. `restart` triggers neither a build nor
a new rollout, so it does not wait.

Recent `Warning` events are printed on any failure (build failed, image pull failed, or timeout)
so you can see *why*.

> Note: a docker `rebuild` that does not change the image produces no new rollout, so there is
> nothing to wait on and it will reach the timeout. Use `image:` to deploy a new tag.

Set `wait: false` to fire-and-forget.

## Setup

1. Create an API token in Hostim.
2. Add it as the `HOSTIM_API_TOKEN` secret in GitHub.

## Versioning

Pin to a major tag and it tracks the latest release in that line:

```yaml
- uses: hostimdev/action@v2   # latest 2.x
```

`@v1` is unchanged and keeps the old fire-and-forget behavior, so existing workflows are not affected by upgrading the repo.

### Migrating from v1

`v2` is backward compatible in its inputs — `action: restart` and `action: rebuild` still work — but the **default behavior changed**:

- In `v1`, a `rebuild` returned as soon as the build was handed off (success = the API accepted it).
- In `v2`, `rebuild` (and any `image` deploy) **waits until the new version is live and fails the job if it doesn't come up**, up to `timeout` seconds.

If you want the old fire-and-forget behavior on `v2`, set `wait: "false"`.
