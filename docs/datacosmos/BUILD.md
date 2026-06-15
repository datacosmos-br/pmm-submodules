# PMM - datacosmos build & fork model

This is a datacosmos fork of [`percona/pmm`](https://github.com/percona/pmm),
tracking the upstream **`v3`** branch.

## Mandatory rule: no bypass, no hidden failure

The datacosmos build must stay minimal over upstream and must never hide a
failure to keep moving. Do not add or keep `|| true`, ignored exit codes,
suppressed validation output, skipped gates, fake success paths, permissive
fallbacks, stubs, synthetic substitutes, or compatibility wrappers in the custom
build, sync, release, or artifact flow. If the real dependency is a daemon,
container, generated file, package repository, image, or SSOT config, make that
dependency work and validate against it instead of replacing it with a narrower
surrogate. Expected optional states must be handled explicitly; missing required
refs, artifacts, images, packages, or credentials must stop the target with a
clear error.

## Branch model

| Branch | Purpose | How to keep current |
|---|---|---|
| `main` | Integration branch - the datacosmos default branch; merges `feat/build` and `feat/clickhouse-collector`. | `git merge upstream/v3` |
| `feat/build` | datacosmos OCI/RPM packaging pipeline (`Makefile.datacosmos`, `datacosmos-release.yml`) + agent coordination protocol. The branch the team builds and releases from. | `git merge upstream/v3` |
| `feat/clickhouse-collector` | Isolated **draft** of a ClickHouse metrics collector. Compilable, not yet upstream-ready. | `git merge upstream/v3` |

Upstream is tracked via the `upstream/v3` remote branch - there is no local
mirror branch. Custom commits live only on the fork branches, so periodic
`git merge upstream/v3` keeps conflicts confined to the fork-specific files.
This mirrors the standard long-lived-fork practice
([Atlassian](https://www.atlassian.com/git/tutorials/git-forks-and-upstreams),
[GitHub Docs](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks/syncing-a-fork)).

## Remotes

```text
origin    https://github.com/datacosmos-br/pmm.git   # the fork
upstream  https://github.com/percona/pmm.git          # Percona
```

## Syncing with upstream (periodic)

```bash
make dc-upstream                         # merge upstream/v3 head
make dc-upstream UPSTREAM_TARGET=stable  # merge the latest stable vX.Y.Z tag
make dc-bump-head                        # pin datacosmos pmm-submodules v3 head
make dc-bump-stable                      # pin datacosmos pmm-submodules pmm-X.Y.Z
make dc-status                           # show Git-derived refs, tags, versions, and worktree status
```

`dc-status` uses Git refs from `origin` and `upstream` only. It prints the
current branch tracking state, upstream `v3` head, latest stable upstream tag,
the upstream commit currently synced into the branch, the last/next datacosmos
tag, forked submodule source settings, and the immediate `git status`.
`dc-upstream` and `dc-bump-*` also print `git status` after they run so merge
conflicts or submodule pointer changes are visible immediately.

## Releases

datacosmos releases are tagged with the **semver-`dcN`** scheme:
`v<upstream version>-dc<N>` - e.g. `v3.8.0-dc2`: the upstream PMM release the
build tracks (`3.8.0`), and `dcN` the datacosmos build counter on top of it
(`dc1`, `dc2`, ...). The image/RPM version drops the leading `v` (`3.8.0-dc2`);
the git tag keeps it (`v3.8.0-dc2`).

For each upstream version, the datacosmos counter starts at `dc1` and increments
from the highest existing `v<upstream version>-dc<N>` tag. The release helper
fetches tags first, so it accounts for releases already pushed to `origin`.
When the upstream commit used by the branch is exactly a stable `vX.Y.Z` tag,
the release helper uses `vX.Y.Z-dc<N>`. Otherwise it uses
`v3-<upstream commit ISO date>-dc<N>`, with `N` incrementing for that date.

```bash
make dc-next                  # next tag, e.g. v3.8.0-dc1 or v3-2026-06-11-dc1
make dc-release               # creates and pushes that tag
```

The earlier date scheme `v3-<ISO date>-<upstream commit count>` still works if
such a tag is pushed manually - the workflow triggers on both `v*-dc*` and
`v3-*` tags - but `dcN` is the current scheme.

Pushing the tag runs `.github/workflows/datacosmos-release.yml`, which builds
linux/amd64 and linux/arm64 images and creates the GitHub Release only after
both architecture builds pass. The build's internal `PMM_VERSION` (written to
`VERSION`, used for the upstream S3 RPM cache) is derived separately from the
nearest upstream semver tag - it stays a clean `X.Y.Z`.

## Building (datacosmos pipeline)

`Makefile.datacosmos` is included by the root `Makefile`, but only exposes
datacosmos-specific `dc-*` targets. Common targets such as `release`, `check`,
`clean`, and `gen` stay owned by upstream Makefiles.

```bash
make dc-build    # prepare + upstream client/server build + artifacts
make dc-publish  # push the built images to ghcr.io/datacosmos-br
make dc-clean    # remove only datacosmos external build/artifact dirs
```

Local builds default to `IMAGE_ARCH=amd64`; set `IMAGE_ARCH=arm64` only on an
arm64 host or runner so the image tag matches the native package build.

## Validation Integrity

The release and local-test gates are root-cause only. Do not mark a datacosmos
build, test, release, or diagnostic path green by using ignored exit codes,
skipped gates, fake success paths, synthetic substitutes, broad fallbacks,
stubs, or compatibility wrappers. If a gate depends on PMM daemons, Docker
Compose services, ClickHouse matrix nodes, generated files, or fork metadata,
make those real dependencies work and validate against them. Evidence must cite
the command, exit code, and decisive output.

`make env TARGET=dc-test-local` is the devcontainer validation path. It starts
the upstream daemon containers through Docker and runs the real checks/tests.
When Docker is remote (`DOCKER_HOST=tcp://...`), service tests must discover the
daemon host from `PMM_TEST_SERVICE_HOST` or `DOCKER_HOST`, and compose/run
targets may bind published ports to `0.0.0.0` via `PMM_TEST_BIND_HOST` so the
devcontainer can reach them. Local Docker keeps loopback binding by default.
Do not replace this with skips, synthetic services, or narrower unit tests.

For local publishing, set `GHCR_USER` to the GitHub login that owns the token
used for `ghcr.io`; CI uses `GITHUB_ACTOR`.

### Upstream-aligned default

`Makefile.datacosmos` deliberately follows upstream's build strategy where the
fork can safely reuse public Percona infrastructure:

- `RPMBUILD_DOCKER_IMAGE` defaults to upstream's public
  `public.ecr.aws/e7j3v3n0/rpmbuild:3` image from `build/scripts/vars`.
- `SKIP_S3_CACHE` is unset by default, so amd64 server RPMs can reuse
  `s3://pmm-build-cache` anonymously and avoid rebuilding Grafana and other
  heavy components.
- The current datacosmos release workflow publishes linux/amd64 and linux/arm64.
- Images are built locally first by `dc-build` and published to
  `ghcr.io/datacosmos-br` only by the explicit `dc-publish` target.
- Datacosmos source bumps use `https://github.com/datacosmos-br/pmm-submodules.git`;
  that fork pins `pmm-dump` to `https://github.com/datacosmos-br/pmm-dump.git`.

### Local-only mode

For a fully local rebuild, set the overrides explicitly:

```bash
docker build --pull --tag pmm-rpmbuild:local --file build/docker/rpmbuild/Dockerfile.el9 build/docker/rpmbuild
RPMBUILD_DOCKER_IMAGE=pmm-rpmbuild:local SKIP_S3_CACHE=1 make dc-build
```

`make dc-build` materialises the build tree under `$(ROOT_DIR)` (default
`../pmm-build-root`, **outside** the repo).

### Build status

The default release path is the GitHub Actions workflow. Local full builds still
need Docker, submodules, and enough disk for PMM's upstream RPM/image pipeline.

## ClickHouse collector (draft - `feat/clickhouse-collector`)

`agent/agents/clickhouse/{collector.go,config.go}` is a `prometheus.Collector`
draft. It **compiles** (after `go get github.com/ClickHouse/clickhouse-go/v2`)
but is **not wired into pmm-agent** and is **not upstream-ready**. Before any
PR to `percona/pmm` it needs: integration into pmm-agent's exporter framework,
English comments, configuration via `pmm-agent.yaml`, and tests.
It is intentionally kept off `datacosmos/build` to avoid carrying the
`clickhouse-go/v2` dependency into the build before the feature is real.

## Agent coordination

This repo adopts the mandatory multi-agent coordination protocol -
see `.agents/skills/agent-coordination/SKILL.md` and the live ledger
`.agents/coordination/LEDGER.md`. Details in `AGENTS.md` W-9.
