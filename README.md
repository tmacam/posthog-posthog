# posthog-posthog (POC fork)

Proof-of-concept fork of [PostHog/posthog](https://github.com/PostHog/posthog), used to see how hard it is to rebuild the images cascade's self-hosted PostHog uses. Not used in production.

## Branch layout

| Branch | Contents |
|---|---|
| `fork-ops` (default) | Only fork automation: workflows and this README. No upstream code. |
| `main` | Pure mirror of `PostHog/posthog@master`. Only ever fast-forwarded by the sync workflow. Never commit here. |

`fork-ops` is the default branch because scheduled and `workflow_dispatch` workflows only run from the default
branch. This keeps fork automation separate from upstream data, so syncing never conflicts with our files.

## Workflows

- **Sync main from upstream** (`fork-sync-upstream.yml`): daily and on demand. It uses the compare API to confirm
  `main` can be fast-forwarded, fetches only the delta, and pushes it. It fails instead of merging if `main` has
  diverged. After pushing, it disables any upstream workflow that the push registered and cancels the
  push-triggered runs, so upstream CI/CD never runs in this fork.
- **Rebuild cascade PostHog images** (`fork-rebuild-images.yml`): on demand. It rebuilds every image in cascade's
  `deployment/ev2/cascade.posthog/images.lock.json` and pushes them to `ghcr.io/tmacam/posthog-posthog/*`:

  | Upstream image | Built from | Our image |
  |---|---|---|
  | `ghcr.io/posthog/rust-node-container` | fork `posthog-rust-node-container`, `Dockerfile` | `rust-node-container` |
  | `ghcr.io/posthog/posthog` | `Dockerfile` | `posthog` |
  | `ghcr.io/posthog/posthog-node` | `Dockerfile.node` (base repointed to our `rust-node-container`) | `posthog-node` |
  | `ghcr.io/posthog/posthog/{capture,property-defs-rs,feature-flags,hypercache-server}` | `rust/Dockerfile` with `BIN=<name>` | same names |
  | `ghcr.io/posthog/posthog/livestream` | `livestream/Dockerfile` | `livestream` |

  Default is amd64 only on standard hosted runners. Upstream builds multi-arch on Depot.

## Secret

`UPSTREAM_SYNC_TOKEN`: a token with `repo` and `workflow` scopes. `GITHUB_TOKEN` can't push commits that change
`.github/workflows/**`, and upstream commits often do. **Proof of concept: remove this secret when finished.**