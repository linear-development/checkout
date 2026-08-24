# `linear-development/checkout`

Drop-in replacement for [`actions/checkout`](https://github.com/actions/checkout) with automatic retry on network failures and [Blacksmith](https://blacksmith.sh) git mirror cache support.

This repo is public so that runners can find it without hassle.

## Features

- **Retry with backoff** -- Up to 3 attempts with 10s/15s backoff between retries. Stalled connections are aborted after ~30s via `GIT_HTTP_LOW_SPEED_LIMIT`/`GIT_HTTP_LOW_SPEED_TIME` instead of hanging until the step timeout.
- **Blacksmith mirror cache** -- Uses [useblacksmith/checkout](https://github.com/useblacksmith/checkout) for persistent local git mirror on Blacksmith runners via Sticky Disk. Falls back to standard clone on non-Blacksmith environments (e.g. GitHub-hosted runners).

## Usage

```yaml
- uses: linear-development/checkout@<sha>
  timeout-minutes: 5
  with:
    persist-credentials: false
```

## Inputs

All inputs are optional and mirror [actions/checkout](https://github.com/actions/checkout).

| Input | Default | Description |
|-------|---------|-------------|
| `persist-credentials` | `true` | Whether to configure the token with the local git config |
| `fetch-depth` | auto | Number of commits to fetch. `0` for all history. Left unset it resolves per runner: `0` where a git mirror is attached and makes full history free, and `1` elsewhere and for container or sparse-checkout jobs, which do not benefit from it. |
| `ref` | | The branch, tag, or SHA to checkout. When checking out the repository that triggered a workflow, this defaults to the reference or SHA for that event. Otherwise, uses the default branch. |
| `submodules` | `false` | Whether to checkout submodules (`true` or `recursive`) |
| `sparse-checkout` | | Sparse checkout patterns (newline-separated) |
| `token` | `${{ github.token }}` | Token used to fetch the repository |
| `repository` | `${{ github.repository }}` | Repository name with owner |

If more options from actions/checkout are needed, just add them.

## How it works

1. Attempts checkout using `useblacksmith/checkout` with `GIT_HTTP_LOW_SPEED_LIMIT=1000` and `GIT_HTTP_LOW_SPEED_TIME=30` to abort stalled connections quickly.
2. If the first attempt fails, waits 10s and retries.
3. If the second attempt fails, waits 15s and makes a final attempt (which fails the step on error).

On Blacksmith runners, the first successful run hydrates a persistent git mirror. Subsequent runs fetch only deltas from the mirror, significantly reducing checkout time and network dependency.
