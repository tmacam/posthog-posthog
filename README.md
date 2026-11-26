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
  This repository is public to use the larger standard public-repository runners (4 vCPU, 16 GB RAM).
  The main PostHog build serializes BuildKit stages and reports runner resources after disk cleanup.
  The public runner still exhausted memory with 15 GiB RAM and no swap, but reported 121 GiB of free disk.
  The build now adds 16 GiB of swap only after checking that at least 80 GiB of disk is free, leaving
  at least 64 GiB for the build. The private runner's earlier swap retry exhausted its much smaller disk.
  `/mnt` shares the root filesystem on the observed runner, so moving files there would not add capacity.
  Final resource and kernel logs help distinguish memory exhaustion from disk exhaustion on future retries.
  Every image build reports the BuildKit container's cgroup v2 `memory.peak` (RAM, including charged cache)
  and `memory.swap.peak`, when supported by the runner kernel, in the logs and job summary.
  These are separate lifetime peaks, not simultaneous usage or a minimum memory requirement.
  Each image job records disk used on `/` right before checkout (after disk cleanup and, for `posthog`, after
  the swap file is created). The Docker build resource report shows free disk and the additional disk used
  since that baseline after the build. This is a point-in-time delta, not peak usage; no background monitor is used.
  Reports run even after build failures, but memory counters are unavailable if the builder has stopped.
  Set `posthog_only=true` to retry just that image, using `posthog_ref` to pin the original source commit.

## Secret

`UPSTREAM_SYNC_TOKEN`: a token with `repo` and `workflow` scopes. `GITHUB_TOKEN` can't push commits that change
`.github/workflows/**`, and upstream commits often do. **Proof of concept: remove this secret when finished.**