# ADR 0002 — Upstream-sync tombstones (feat-branch evaluation, 2026-06-15)

## Status

Accepted — 2026-06-15.

## Context

The fork was advanced onto upstream base `7cc7d516` (PMM-15144) by an upstream
merge into `sync/tibi-holmes` (the 255-commit datacosmos delta is preserved, per
[AGENTS.md §17]). Six datacosmos feature-branch tips and two upstream commits were
submitted for evaluation with the directive: **accept only real improvements; for
anything discarded, leave a tombstone**.

Two were kept (already integrated): `0a7452c49` (docs dc-* targets, = HEAD parent)
and `22fbb58da` (the `upstream/v3` merge, integrated as `9b070f3c8`, conflict
markers resolved in commit `65c31a2bc`).

This ADR records the **discarded** commits with the decisive read-only evidence
proving each is already represented in HEAD or superseded — so no functionality is
lost and the discards are auditable. Evidence captured against HEAD `40e0d0bf2`.

## Decision — tombstoned commits

| Commit | Branch | Subject | Verdict | Decisive evidence (HEAD `40e0d0bf2`) |
|--------|--------|---------|---------|--------------------------------------|
| `e1d9cbcd6` | `feat/upstream-sync` | fix(pmm): support Grafana 12 dashboard build | **Already represented** | `git diff --quiet HEAD e1d9cbcd6 -- <file>` rc=0 for **all 6** touched files (`pmm-qan/.../BarChart.utils.ts`, `.../Metrics.utils.ts`, `.../Latency.styles.ts`, `Form/FieldAdapters/Checkbox.tsx`, `Form/FieldAdapters/Field.tsx`, `shared/.../getPmmTheme.ts`) — byte-identical in HEAD. Cherry-pick = no-op. |
| `5265d2ad7` | `feat/v3.8.0-dc-stable` | build: keep client binary script executable | **Already represented** | `git ls-files -s build/scripts/build-client-binary` → mode `100755` in HEAD. Exec bit already set. |
| `2bc773c4c` | `feat/build` | ci(release): pre-pull the rpmbuild image with retries | **Superseded** | `.github/workflows/datacosmos-release.yml` already contains the pre-pull/retry logic (`grep -c 'Pull the rpmbuild image\|toomanyrequests\|retries'` → 3). HEAD's workflow is ahead of the commit's older tree. |
| `98384d2dc` | `feat/upstream-sync-pmm-submodules` | docs: update argo-cd/ path in managed/AGENTS.md | **Superseded — would regress** | HEAD has `managed/AGENTS.md` (current). The commit's diff would revert HEAD doc improvements (`../AGENTS.md`→`../../AGENTS.md`, re-add Copilot frontmatter, revert `go:generate` line). Not applied. |

## Decision — upstream commits

| Commit | Subject | Verdict | Decisive evidence |
|--------|---------|---------|-------------------|
| `7cc7d516` | PMM-15144 self-heal Grafana token + annotations layer | **Integrated** | Merged into `sync/tibi-holmes` (HEAD reachable); union conflicts resolved in `65c31a2bc`; `make check` exit 0. |
| `2222a5d4` | Refactor ClickHouse: replace built-in disable flag with host address checks | **Not applicable — fork supersedes** | The fork independently implements the same "host address check" design, more cleanly at the model layer: `ClickHouseParams.ExternalClickHouse() = !internalAddr(p.url.Hostname())` (`managed/models/clickhouse_params.go:32`). The upstream `clickhouseBuiltinDisabled` flag does not exist in the fork (`grep -rn 'clickhouseBuiltinDisabled' managed/` → 0 hits). Porting `2222a5d4` would conflict against superior fork code. |

## Consequences

- No functionality lost: every discarded commit is proven already-present or
  superseded; the upstream `2222a5d4` behavior is delivered by the fork's
  `ExternalClickHouse` model method.
- This file lives under `docs/datacosmos/` so it survives future upstream syncs.

[AGENTS.md §17]: ../../../AGENTS.md
