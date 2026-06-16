# ADR 0001: No Bypass In Datacosmos Build Governance

## Status

Accepted

## Context

The datacosmos PMM fork carries a small custom layer over upstream PMM for
versioning, forked submodule refs, amd64 release builds, GHCR publishing, and
GitHub Releases. Previous automation allowed some failures to continue through
`|| true`, success exits on fatal paths, or skipped artifact checks. That hides
the real failure source and can publish an incomplete or unverifiable release.

## Decision

The strongest rule for datacosmos build governance is: fix the root cause at
the source and fail visibly. Custom build, sync, status, artifact, package, and
release paths must not use `|| true`, ignored exit codes, suppressed validation
output, fake success paths, permissive fallbacks, stubs, synthetic substitutes,
or compatibility wrappers to turn a failed operation into a green result. If
the real dependency is a daemon, container, generated file, package repository,
image, or SSOT config, that dependency must be made functional and validated
directly instead of being replaced by a narrower surrogate.

Expected optional states must be represented with explicit branches. Required
single sources of truth, refs, artifacts, credentials, package outputs, release
assets, and validation gates must stop the workflow with a clear error when
missing or invalid.

## Consequences

- Release automation fails earlier when required inputs or artifacts are absent.
- Diagnostic commands can report optional missing state, but they cannot hide
  failed Git, build, or release operations.
- Completion evidence for build-governance changes must include command, exit
  code, and decisive output.
- Any future exception requires replacing this ADR with a stricter root-cause
  design, not adding a local bypass.
