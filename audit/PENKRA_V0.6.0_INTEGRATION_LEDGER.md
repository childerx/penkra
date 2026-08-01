# Penkra / Penkra v0.6.0 Integration Ledger

> Historical integration record: this ledger describes the v0.6.0 integration at the time it was
> completed. The separate Canary flavor recorded below was subsequently retired for the v0.8.0
> release line; Stable and Penkra Dev are now the only desktop identities.

## Fixed boundary

- Client scope: `Penkra Internal` (`00000000-0000-4000-8000-000000000000`), verified with
  `penkra whoami`.
- Penkra starting commit: `fb692c3781743011c3c5c8432ef9619c6fee7e29`.
- Integrated Penkra baseline before this work: `v0.5.5`.
- Penkra release: annotated tag `v0.6.0`; tag object
  `a4050a72679b703346dedddf6aa2af0d18ba7fe0`; peeled commit
  `ea136916bf558936cc1f10ed5665df563efbbebb`.
- Upstream delta: 153 commits; 1,190 changed files; 178,894 insertions; 78,220 deletions.
- Method: merge the tagged implementation first, including its tests and migrations, then adapt or
  prune against verified Penkra behavior.
- Integration branch/worktree: `integrate/penkra-v0.6.0` in
  `penkra-console-penkra-v0.6.0`.
- The operator approved the detailed integration proposal and then directed implementation through
  production deployment. Publication remains gated on the final quality suite and artifact checks.
  `PENKRA.md` was not changed.

## Complete scope accounting

| Area                               | Decision                        | Result                                                                                                                                                                                                                                                                                                                                                                                |
| ---------------------------------- | ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Agent Gateway and delegation       | Import, then adapt              | Imported the upstream gateway, durable operations, recovery, diagnostics, wait/read/send flows, and provider injection. Restored Penkra host policy and client/project authority. Removed Automation tools and added negative tool-surface checks.                                                                                                                                    |
| Project Spaces                     | Import and enable               | Imported migration 79, projections, routes, shortcuts, grouping UI, stores, and tests. Spaces group only ordinary projects present in the active client read model; missing/out-of-scope projects and dangling spaces are rejected. Registry-backed Penkra project labels survive.                                                                                                    |
| Cross-task mentions                | Import                          | Imported thread mentions and source-aware presentation so an operator can reference another authorized Penkra task without copying its conversation.                                                                                                                                                                                                                                  |
| Providers and subagents            | Import, then adapt              | Imported the v0.6.0 provider/ACP/Codex improvements. Centralized Penkra provider environment injection restores the active workspace config and thread identity without inheriting stale credentials.                                                                                                                                                                                 |
| Durable orchestration and recovery | Import                          | Imported command receipts, runtime events, queued-turn durability, diagnostics, checkpoint coordination, attachments, handoff, startup recovery, and reconciliation changes.                                                                                                                                                                                                          |
| Performance                        | Import                          | Imported SQLite WAL mode with `synchronous=NORMAL`, projection/index work, bounded streams, and the upstream refactors. Increased the bounded macOS process-snapshot buffer after the full suite proved 256 KB could not establish provider-process exit on a busy machine.                                                                                                           |
| Complete UI/UX                     | Import, then adapt              | Kept the upstream web/desktop interaction, accessibility, layout, style, route, browser, composer, terminal, pull-request, settings, and Space changes. Reapplied Penkra branding, client project names, `$` skills, todo/profile/top-right surfaces, root picker, and HQ behavior.                                                                                                   |
| Security and recovery              | Import                          | Kept origin/auth hardening, bounded request handling, attachment ownership, process-tree exit proof, updater hardening, migration backup/recovery, and bundle-swap protection. Updated trusted desktop origins to `penkra://app` and `penkra-canary://app`.                                                                                                                           |
| macOS desktop/AppSnap/updater      | Import, then adapt              | Kept upstream macOS implementation as the base. Restored `com.penkra.app`, Penkra/Canary storage, private token-authenticated S3 updates, resumable and differential downloads, final ZIP blockmap regeneration, bundled Penkra backend CLI, and legacy `penkra://` storage import. Production is macOS arm64; Windows/Linux release lanes and package commands are absent.           |
| External MCP                       | Import for analysis, then prune | Removed contracts, runtime, HTTP/WS routes, settings/UI, pairing, tools, migrations 74–78/80, tests, and docs. Cleanup migration 80 drops all legacy External MCP tables. Only cleanup evidence remains.                                                                                                                                                                              |
| Penkra Automations                 | Import for analysis, then prune | Removed repositories, scheduler/runtime/reactors, contracts, routes, commands, settings/UI, work-log markers, Agent Gateway tools, plans, and release-note feature cards. Historical Penkra migration ids 44–48 remain only so existing databases keep valid lineage; migration 80 removes their tables and view. The legacy dispatch-origin literal remains only to decode old rows. |
| Windows/Linux-only work            | Import for analysis, then prune | Removed live distribution commands, release workflow lanes, build configuration, platform UI, and platform documentation. Shared utilities still required by macOS are retained; the production release smoke explicitly denies Windows/Linux lanes.                                                                                                                                  |
| Penkra identity/public publishing  | Replace or prune                | User-facing identity is Penkra, production bundle identity is `com.penkra.app`, Canary is isolated, and publishing is the existing private S3 flow. Internal `@penkra/*`, `penkra_*` gateway protocol names, database lineage, and the legacy scheme remain where renaming would break compatibility rather than change product identity.                                             |

## Penkra behavior preserved

- One active client at a time, registry synchronization, isolated workspace ownership, HQ
  authentication, project pinning, and registry-derived project names.
- Server-backed `$` skills and Penkra instructions.
- Provider `PENKRA_CONFIG`, `PENKRA_SESSION_ID`, project working directory, and stale-secret
  stripping for Codex and ACP-derived providers.
- Todo panel and badges, profile pane, top-right panel, Penkra project presentation, desktop root
  picker, backend client/socket/scaffold, and bundled backend CLI.
- Private updater authentication, resumable downloads, differential blockmaps, release version
  pinning, and recoverable local installation.

## Migration and compatibility result

- The imported v0.6.0 migration lineage is retained through migration 79 (`Spaces`).
- Migration 80 (`PruneRejectedPenkraSurfaces`) removes every Automation and External MCP table and
  the legacy Automation completion-evaluation view while retaining `projection_spaces`.
- Existing renderer data is migrated once from the legacy `penkra://` origin to `penkra://`; the
  completion marker is written only after the renderer acknowledges the validated snapshot.
- SQLite requests WAL mode and `synchronous=NORMAL`; in-memory test databases truthfully report
  that WAL is unavailable rather than pretending it was enabled.

## Verification ledger

| Gate                        | Evidence                                                                                                                                                                                                                                                                                                                                 | State                                 |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| Merge integrity             | `MERGE_HEAD` is the signed v0.6.0 tag object; the tag peels to `ea136916...`; `git diff --name-only --diff-filter=U` is empty.                                                                                                                                                                                                           | Complete                              |
| Migration lineage/cleanup   | `Migrations.test.ts` plus historical 44 cleanup: 17/17 passed.                                                                                                                                                                                                                                                                           | Complete                              |
| Contracts                   | 12 files, 117 tests passed.                                                                                                                                                                                                                                                                                                              | Complete                              |
| Agent Gateway               | Focused gateway suite: 69 passed; excluded tool names asserted absent.                                                                                                                                                                                                                                                                   | Complete                              |
| Spaces/client boundary      | Focused Spaces suite: 12/12 passed, including rejection of a project absent from the active client read model.                                                                                                                                                                                                                           | Complete                              |
| ACP/provider lifecycle      | ACP JSON-RPC and official SDK suites: 27/27 passed after bounded process-snapshot repair.                                                                                                                                                                                                                                                | Complete                              |
| Penkra conflict regressions | Codex environment, CLI surface, trusted origins, local server filtering, harness policy, profile identity, tool labels, release notes, and project/skill presentation focused suites passed.                                                                                                                                             | Complete                              |
| Manual macOS QA             | In Penkra (Dev), verified client avatars and badges, no HQ avatar action, Profile opening in the right dock, switching the Profile to another client, Profile tab/close controls, and top-right panel controls. Registry-backed Add Client and client Instructions could not be exercised because the isolated Dev registry was offline. | Complete with noted environment limit |
| Release policy              | `bun run release:smoke` passed for manual macOS arm64 build, pinned backend CLI, private update token, and opt-in S3 publication.                                                                                                                                                                                                        | Complete                              |
| Brand identity              | `bun run brand:check` passed.                                                                                                                                                                                                                                                                                                            | Complete                              |
| Full repository tests       | `bun run test`: 8/8 workspace tasks succeeded; 6,343 tests passed and 13 intentionally skipped across 608 passing test files and 3 skipped files.                                                                                                                                                                                        | Complete                              |
| Full build                  | `bun run build`: 5/5 workspace build tasks succeeded.                                                                                                                                                                                                                                                                                    | Complete                              |
| Patch integrity             | `git diff --check` passed; no unresolved merge paths.                                                                                                                                                                                                                                                                                    | Complete                              |

## Final diff accounting

- Relative to the v0.6.0 commit, 527 tracked paths differ: 52 added, 98 deleted, and 377 modified.
  These are the explicit Penkra adaptations and pruned surfaces above, not a second implementation
  of upstream ideas.
- Relative to the Penkra starting commit, the uncommitted integration changes 1,209 tracked paths
  (162,286 insertions and 102,406 deletions) before counting the new cleanup migration and this
  ledger.
- Every Penkra-added path in the pre-integration divergence inventory still exists. No unresolved
  textual merge path remains.
- The original dirty Penkra worktree was not used for integration and was not modified by this
  work.

## Approval and release boundary

The operator approved the proposed scope, directed implementation, requested manual QA before
continuing, and then directed work through a production update. Commit, reconciliation, build, and
publication may proceed only after the recorded gates pass on the exact release tree. Do not change
`PENKRA.md`.
