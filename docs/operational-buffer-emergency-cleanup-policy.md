# Operational Buffer & Emergency Cleanup Policy — 256GB Mac

**Status:** Accepted architecture/operations policy for Issue #9  
**Date:** 2026-09-04  
**Scope:** Define thresholds, triggers, risk classes, and execution guardrails only. This document does not authorize deleting, moving, installing, reformatting, or reconfiguring real user data or applications.

## 1. Policy objective

This policy converts the storage-safety findings from Issue #5 and the Storage Tiering decision from Issue #6 into an actionable operating model for the current 256GB-class Apple Silicon / 8GB Mac.

The goals are:

- prevent the internal SSD from repeatedly falling into a critically low-space state;
- define when large writes, installs, and updates must stop;
- distinguish emergency recovery from normal housekeeping;
- classify cleanup candidates by risk;
- require explicit user authorization for every real mutation;
- stop cleanup once sufficient headroom has been restored instead of pursuing cosmetic maximum free space.

## 2. Authority and evidence basis

This policy carries forward the accepted upstream conclusions:

- **Issue #5:** current internal available capacity around 2.6–2.9GiB is a Critical / Emergency condition; approximately 12–15GiB free is the Phase 1 recovery target; approximately 20–25GiB free is the Phase 2 healthy steady-state target. These are project engineering targets, not universal Apple thresholds.
- **Issue #6:** Internal SSD is the Control Plane / Hot Core; Samsung T7 is the Warm Capacity / Large Working Set; cloud is Cold / On-demand Archive; backup is orthogonal.
- **Issue #2:** Downloads, application data, caches, developer tooling, and user data all contribute to pressure; duplicate candidates and historical snapshots exist but are not automatically safe to delete.
- **Issue #11:** some large creative applications/assets are external-placement candidates, but system/license roots and unsupported relocation hacks must stay protected.

## 3. Capacity states

These states are **project operating policy for this machine**, not Apple-published universal limits.

| State | Internal free space | Operating interpretation | Default policy |
|---|---:|---|---|
| **RED — Critical / Emergency** | **< 5GiB** | Immediate risk from insufficient room for swap/temp/log/cache/update staging | Freeze large writes; recovery work only |
| **ORANGE — Recovery Required** | **5–12GiB** | System can operate, but headroom remains fragile | Continue controlled recovery; no large installs/updates |
| **YELLOW — Minimum / Constrained** | **12–20GiB** | Basic daily work acceptable, but capacity governance still active | Resume ordinary light work; defer unnecessary high-write tasks |
| **GREEN — Healthy Working** | **20GiB+** | Normal target operating range for this machine | Routine work allowed; continue lifecycle discipline |

### Phase targets

- **Phase 1 recovery target:** approximately **12–15GiB free**.
- **Phase 2 steady-state target:** approximately **20–25GiB free**.

Reaching Phase 1 is sufficient to exit emergency recovery. Reaching Phase 2 is sufficient for normal steady-state operation. Cleanup should stop unless a separate task justifies additional reclamation.

## 4. Global operating gates

### RED — Critical / Emergency

Allowed by default:

- read-only inspection and size measurement;
- text/code review;
- planning and generation of cleanup candidates;
- explicitly authorized recovery actions.

Blocked by default:

- large downloads;
- large archive extraction;
- new large application installs;
- macOS updates/upgrades;
- large rendering/baking/simulation jobs;
- speculative bulk moves;
- reformatting/partitioning external storage.

### ORANGE — Recovery Required

Allowed:

- light daily browsing/document/code work;
- Git/text operations;
- explicitly authorized cleanup/migration batches.

Still blocked:

- large application installs;
- macOS updates unless a separate preflight proves sufficient headroom;
- high-write creative workloads when avoidable;
- unplanned downloads/archive extraction.

### YELLOW — Minimum / Constrained

Allowed:

- ordinary daily work;
- small tools and low-write workflows;
- normal project activity with monitoring.

Still requires preflight:

- large application installs;
- OS updates/upgrades;
- large creative caches/renders;
- any operation expected to materially reduce free space.

### GREEN — Healthy Working

Normal work is allowed, but GREEN is not a blanket authorization for any installer/update. Large installs and OS updates still require candidate-specific preflight using vendor/system-reported requirements and current free-space measurement.

## 5. Install / update preflight policy

There is no single universal “install-ready” number.

Before any large install or OS update:

1. measure current free space using a machine-readable source such as POSIX `statvfs`/`df` or Foundation `volumeAvailableCapacityKey`;
2. obtain the installer/update's own reported storage requirement when available;
3. identify whether the operation uses startup-volume staging/temp space;
4. ensure the operation is not expected to push the system back into RED/ORANGE;
5. if the required peak/staging space is unknown, treat that uncertainty as a reason to defer rather than assume GREEN alone is sufficient.

Substance Painter remains separately gated by Issue #12 because the current machine's 8GB RAM is below Adobe's published macOS minimum.

## 6. Cleanup risk classification

Risk class describes **candidate priority and review depth**, not authorization.

**All real mutations require explicit user authorization in the execution ticket.**

### LOW RISK — eligible for first recovery batch after verification

A candidate may be LOW RISK only if all of the following are true:

- data is reproducible, disposable, or vendor-documented as cache/temp;
- it is not the only copy of user-authored or downloaded content;
- it is not a database, message history, library catalog, sync root, license state, or active project source;
- the application is not actively using it;
- the expected reclaim is measured before action;
- the cleanup mechanism is documented/native when possible.

Typical examples **after local verification**:

- clearly disposable application caches with vendor/native cleanup support;
- package/download caches that can be deterministically rebuilt or re-downloaded;
- known temporary/staging files left by completed tasks;
- regenerated render/build intermediates that are confirmed non-authoritative.

LOW RISK means “good first candidate for a user-approved batch,” **not** “Agent may delete automatically.”

### REVIEW FIRST — common space-recovery candidates requiring item-level judgment

Examples:

- Downloads contents;
- DMG/PKG/ZIP installers;
- old application versions;
- duplicate-copy candidates;
- inactive project folders;
- old Unity Editor versions/modules;
- large `Application Support` subdirectories whose semantics are not proven cache-only;
- Trash contents;
- local media/download folders from cloud clients;
- moving projects/apps/assets from internal SSD to T7;
- archiving data to cloud followed by removal of the local source;
- old developer environments, SDKs, package stores, or Conda environments.

Required before mutation:

- identify what the item is;
- determine whether it is authoritative or reproducible;
- estimate reclaim;
- define rollback/recovery path;
- verify destination/second copy when moving or archiving;
- obtain explicit user authorization.

### HIGH RISK — never part of automatic emergency cleanup

Examples:

- wholesale `~/Library` or `/Library` deletion/migration;
- Adobe/Creative Cloud licensing, SLCache/SLStore, or system-level support roots;
- Mail, Messages, Photos, Contacts, browser profile databases, or communication app databases;
- Zotero/Calibre/knowledge libraries when copy/backup status is not verified;
- user-authored source projects, teaching material, research material, recordings, or unique personal data;
- cloud-sync metadata/databases or existing Baidu Netdisk assets;
- macOS system directories, VM/swap files, Preboot/Recovery/System volumes;
- APFS snapshot deletion without a separate verified reason;
- Homebrew/Conda/Node/Python runtime relocation to the current exFAT T7 before #4/#15;
- T7 reformat/partition changes;
- ad-hoc cross-volume symlink hacks for system/application roots;
- deleting a source because a cloud upload merely returned success.

HIGH RISK actions require a dedicated scoped execution decision and explicit user approval; many should remain prohibited unless a strong recovery plan exists.

## 7. Emergency cleanup execution protocol

Issue #13 will implement this protocol when authorized.

### Step 1 — Baseline

Record before mutation:

- current internal free space;
- current storage state (RED/ORANGE/YELLOW/GREEN);
- T7 mount health if T7 will be used;
- candidate size and location in local-only evidence.

### Step 2 — Candidate table

For each candidate record:

- risk class;
- estimated reclaim;
- authoritative vs reproducible status;
- planned action (delete / move / archive / native cache clear);
- rollback/recovery method;
- verification method.

### Step 3 — User authorization

Present one small batch at a time. Authorization must identify the candidate class/action clearly enough that the user knows what will change.

No mutation occurs from this policy alone.

### Step 4 — Execute smallest useful batch

Prefer LOW RISK candidates first. Do not bundle unrelated REVIEW FIRST actions into a broad “clean everything” command.

### Step 5 — Verify immediately

After each batch:

- verify command/result;
- measure actual reclaimed space;
- check affected app/project if relevant;
- verify destination/hash before considering source removal for migrations;
- record any local-only side effects.

### Step 6 — Sufficiency Stop

- If Phase 1 target (~12–15GiB) is reached, stop emergency recovery unless the user explicitly chooses to continue.
- Do not chase Phase 2 in the same emergency batch merely because more cleanup is possible.
- Phase 2 (~20–25GiB) should normally be achieved through planned tiering/workspace changes rather than aggressive deletion.

## 8. T7-specific safety gates

Before any write/move to T7:

- verify the expected T7 mount is actually mounted;
- do not rely only on a path string under `/Volumes`;
- abort if the mount is absent or differs from expected;
- never create a same-named fallback directory on the internal SSD;
- for important migrations, verify copied content before source deletion;
- remember that T7-only is still one copy, not a backup.

Because the current T7 is exFAT, POSIX-sensitive development runtimes remain excluded until #4/#15.

## 9. Cloud archive safety gates

For `/apps/bdpan/` or another cloud archive:

- uploads must be explicit and scoped;
- important bundles should use manifest/hash verification when practical;
- successful upload does not automatically authorize local deletion;
- existing Baidu user assets outside the bounded Skill scope are not implied to be accessible or mutable;
- cloud archive is not treated as complete disaster recovery.

Backup/Restore policy is tracked separately in #14.

## 10. Monitoring / trigger semantics

If later automated monitoring is added, it may:

- measure free space;
- classify current state;
- notify when thresholds are crossed;
- prepare candidate reports.

It must **not** automatically delete, move, empty Trash, clear app data, archive-and-delete, or alter storage configuration solely because a threshold was crossed.

Suggested project alert semantics:

- **RED:** `<5GiB` — emergency notice; block large writes and prepare recovery.
- **ORANGE:** `5–12GiB` — recovery remains priority.
- **YELLOW:** `12–20GiB` — warn before high-write operations; restore healthy buffer over time.
- **GREEN:** `20GiB+` — normal operation.

The exact automation implementation belongs to a later ticket; these thresholds are project policy inputs.

## 11. Current machine decision

At the latest verified/reported probe, internal free space is approximately 2.6–2.9GiB.

Therefore the machine is currently **RED — Critical / Emergency**.

Current operational directive:

- do not install Painter;
- do not run macOS update;
- do not start large downloads/extractions;
- do not start unplanned high-write creative jobs;
- proceed next to Issue #13 **planning**, using the LOW RISK / REVIEW FIRST / HIGH RISK model above;
- any actual cleanup/move/delete remains blocked on explicit user authorization.

## 12. Decision

**Accepted:**

> Space thresholds trigger behavior changes, not autonomous deletion. Emergency recovery proceeds through measured, reversible, user-authorized small batches; LOW RISK determines priority, not permission. Recovery stops once sufficient operating headroom has been restored.

This is sufficient to close Issue #9 and unblock Issue #13 for execution planning only.