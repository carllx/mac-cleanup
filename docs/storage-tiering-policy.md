# Storage Tiering Policy — 256GB Mac / Samsung T7 / Cloud Archive

**Status:** Accepted architecture decision for Issue #6  
**Date:** 2026-09-04  
**Scope:** Decide storage responsibilities only. This document does not authorize moving, deleting, installing, reformatting, or reconfiguring user data or applications.

## 1. Decision summary

This project adopts a **three-tier storage model plus an orthogonal backup role**:

1. **Tier 0 — Internal SSD = Control Plane / Hot Core**
   - Keep macOS, launchers, license services, system-level Application Support, small/high-frequency applications, and a small working set that must remain usable when the external drive is disconnected.
   - Preserve internal free-space headroom as a first-class resource, not as leftover capacity.

2. **Tier 1 — Samsung T7 = Warm Capacity / Large Working Set**
   - Use the T7 as the primary capacity expansion layer for large active projects, large asset libraries, verified portable/self-contained creative applications, versioned Editors/modules, media/render/scratch data, and other large data that can tolerate external-drive dependency.
   - Because the current T7 is exFAT, do **not** place POSIX-sensitive development runtimes there by default.

3. **Tier 2 — Cloud Archive = Cold / On-demand Archive**
   - Use cloud storage for selected cold bundles, historical deliverables, teaching archives, and secondary archive copies that do not need to remain locally mounted.
   - The runtime-proven Baidu path is the bounded `/apps/bdpan/` Agent-managed archive zone, operated through `baidu-drive` / `bdpan` on explicit transfer actions.
   - Do not use Baidu desktop full-sync as the primary storage layer for this 256GB Mac.

4. **Backup is not another storage tier**
   - Sync, archive, and backup are different responsibilities.
   - A T7-only file is one copy, not a backup.
   - A two-way sync mirror is not automatically a backup because deletion/corruption can propagate.
   - `/apps/bdpan/` is accepted as a bounded secondary archive candidate, not as a complete disaster-recovery system.
   - High-value, irreplaceable data requires an independent Backup / Restore policy; follow-up is tracked separately.

## 2. Authority and evidence basis

This decision integrates the completed upstream work:

- **#2 Storage topology:** internal SSD is critically constrained; T7 already carries large/historical assets; teaching/code/media are scattered across tiers; some knowledge/high-value data have single-copy risk; Downloads has become an unmanaged pseudo-workspace.
- **#3 Sync/archive research:** separate storage backend from Agent automation; separate sync from archive and backup; rclone is a strong general transfer engine; Baidu desktop full-sync is a poor fit for this machine.
- **#10 Baidu runtime prototype:** `baidu-drive` / `bdpan` completed OAuth → isolated upload → structured list → download → SHA-256 round trip inside `/apps/bdpan/`.
- **#11 Application placement:** Unity Editor/PlaybackEngines and Blender/application assets are strong T7 candidates; Creative Cloud core and Adobe licensing stay internal; Painter Libraries and Temp/SVT cache can be external; Painter binary remains a separate compatibility/install question.
- **#5 macOS storage margin:** current internal free space around 2.6–2.9GiB is Critical / Emergency; project engineering targets are approximately 12–15GiB for Phase 1 recovery and 20–25GiB for normal healthy operation. These are project policy targets, not Apple universal thresholds.

## 3. Tier 0 — Internal SSD policy

### Keep on internal by default

- macOS and Apple system volumes.
- Login items, launchers, updaters, license/activation services.
- Creative Cloud Desktop and Adobe core licensing/support components.
- Unity Hub and comparable lightweight orchestration services.
- Small, frequently used applications where external placement provides little space benefit.
- Small active documents/code/config needed for basic work when T7 is unavailable.
- System/user state whose vendor does not provide a supported relocation mechanism.
- Temporary installation/update staging that macOS or the vendor requires on the startup volume.

### Capacity policy

- **Current state:** Critical / Emergency at approximately 2.6–2.9GiB available.
- **Phase 1 recovery target:** approximately 12–15GiB free.
- **Phase 2 steady-state target:** approximately 20–25GiB free.
- When below the project emergency threshold, do not start large installs, OS updates, large downloads, archive extraction, heavy baking/rendering, or other high-write tasks.

### Internal SSD is not the default warehouse

Large projects, media, historical teaching packages, large assets, and relocatable application content should not remain on internal storage merely because that is the default installer/download location.

## 4. Tier 1 — Samsung T7 policy

### Strong candidates

- Unity Editor versions and PlaybackEngines/modules already verified in external placement.
- Large Unity projects and similar large creative projects.
- Blender application/portable configuration where appropriate.
- Blender asset libraries and project data.
- Substance/Painter Libraries, assets, Temp/SVT cache when/if Painter is used.
- Media source files, render output, simulation/bake/scratch data.
- Historical teaching/media/project packages that still need relatively fast local access.
- Large application-specific caches **only when the application provides a native supported path setting**.

### T7 must not become an unstructured dumping ground

T7 is a **warm working tier**, not an infinite Downloads folder. Issue #7 will define the durable workspace/directory architecture and active → archive lifecycle.

### exFAT constraint

The current T7 is exFAT. Therefore:

- Ordinary documents, media, assets, project bundles, and verified self-contained creative applications are acceptable candidates.
- Do not move Homebrew roots, Conda environments, Node/Python runtimes, Unix-heavy toolchains, container data, package stores, or other POSIX-sensitive workloads to T7 by default.
- Do not solve compatibility problems with ad-hoc cross-volume symlink hacks.
- Development-environment placement waits for #4 and the follow-up runtime validation ticket.

## 5. Tier 2 — Cloud archive policy

### Baidu role

Accepted role:

- `/apps/bdpan/` = **bounded Agent-managed archive zone**.
- Suitable for explicitly selected archive bundles, historical deliverables, or secondary copies.
- Transfers should be explicit, observable, and verifiable; hash/manifest verification is preferred for important bundles.

Not accepted:

- Baidu desktop full-sync as the primary working layer on the internal SSD.
- Treating the whole existing Baidu Netdisk as automatically available to the Agent.
- Treating `/apps/bdpan/` as a complete versioned/immutable backup system.
- Deleting the local/source copy merely because an upload command returned success; source deletion requires a separate verified migration/retention decision.

### Other cloud backends

rclone remains the preferred general-purpose automation engine if a future object-storage or other supported backend is chosen. No new cloud service is selected or purchased by this decision.

## 6. Placement matrix

| Workload / component | Default tier | Notes |
|---|---|---|
| macOS / system services | Internal | Mandatory hot core |
| License / launcher / updater services | Internal | Keep independent of T7 availability |
| Small/high-frequency applications | Internal | Low space benefit from externalization |
| Unity Editor / PlaybackEngines | T7 | Verified external candidate |
| Unity large projects | T7 | Warm working set |
| Blender.app / portable setup | T7 candidate | Use supported placement; not urgent for 802MiB current binary |
| Blender assets / large projects | T7 | Strong candidate |
| Creative Cloud core / Adobe licensing | Internal | Do not cross-volume symlink |
| Painter app binary | No placement approval yet | #12 gates practical compatibility; external app install is not established |
| Painter assets / Libraries / Temp/SVT | T7 candidate | Native relocation mechanisms exist |
| Large media / render / bake / scratch | T7 | Keep high-write bulk off internal |
| Active small docs / small code repos | Internal by default | Maintains basic continuity when T7 is absent |
| Large code/media repos | Case-by-case | T7 only if filesystem semantics are safe; dev runtime awaits #4 |
| Historical teaching/project bundles | T7, then Cloud archive as needed | Warm → cold lifecycle |
| Downloads | Staging only | Not a durable storage tier; #7 will reorganize |
| Cold archive bundles | Cloud archive | Explicit upload/verify, not full sync |
| Irreplaceable primary data | Any primary tier + independent second copy | Backup policy required |

## 7. Disconnection and degradation policy

### If T7 is disconnected

The Mac must still be able to:

- boot and log in;
- run core communication/browser/IDE/text workflows;
- retain licensing/launcher state;
- access a minimal current working set.

T7-dependent creative applications/projects may become unavailable until reconnect. This is an accepted degradation mode.

Any automation configured to write to T7 must verify that the expected mount is actually present before writing. It must not silently create a same-named ordinary directory under `/Volumes` on the internal disk.

### If cloud is unavailable

Current internal/T7 work continues. Archive upload/download is deferred. Cloud availability must never be required for basic local operation.

## 8. Sync vs archive vs backup

### Sync

Purpose: keep multiple working copies converged.

Policy: not the default mechanism for large data on this capacity-constrained Mac. Two-way sync is never treated as backup by itself.

### Archive

Purpose: move or copy inactive/cold material out of the hot working set while retaining retrievability.

Policy: T7 can hold warm archives; `/apps/bdpan/` can hold selected cold/secondary archive bundles.

### Backup

Purpose: recover from device loss, corruption, mistaken deletion, account loss, or destructive changes.

Policy: orthogonal to tiering. A primary item is not considered protected merely because it resides on T7 or participates in sync. High-value data needs a separately designed, independently recoverable copy strategy. Follow-up: Issue #14.

## 9. Migration order — policy only, not execution authorization

### Phase 0 — Emergency freeze

While internal free space remains near the current Critical level:

- no Painter install;
- no macOS update;
- no large download/extraction;
- no speculative bulk migration;
- no reformatting T7.

### Phase 1 — Recover operating headroom

Target: approximately 12–15GiB internal free space.

Execution candidates must be selected from local-only facts and classified by risk. Prioritize:

1. low-risk reproducible/cache/staging data identified by #9;
2. data already proven to belong on T7 but still occupying internal space;
3. clearly inactive large files suitable for verified relocation/archive;
4. duplicates only after identity/retention verification.

The actual execution is tracked in #13 and requires explicit user approval before any mutation.

### Phase 2 — Establish healthy steady state

Target: approximately 20–25GiB internal free space.

Use the tier rules in this document plus the workspace lifecycle from #7. Do not keep reclaiming space past the healthy target merely for cosmetic cleanliness.

## 10. Guardrails / non-goals

- Do not move all of `~/Library`.
- Do not symlink Adobe/system roots to T7.
- Do not move POSIX-sensitive development runtimes to exFAT before #4/#15 evidence.
- Do not turn Baidu desktop full-sync into the primary storage architecture.
- Do not treat T7-only or Baidu-only storage as sufficient backup for irreplaceable data.
- Do not delete a source immediately after archive upload without a separate verification/retention gate.
- Do not install Painter until #12 resolves the 8GB RAM compatibility boundary.

## 11. Follow-up tracker

- **#7** — define Target Workspace Architecture and active → warm → cold lifecycle.
- **#9** — define Operational Buffer & Emergency Cleanup Policy.
- **#13** — execute the first controlled internal-space recovery after #6 + #9, with explicit user approval.
- **#14** — define Backup / Restore Policy independently from sync/archive.
- **#4 / #15** — inventory and validate development-environment placement under exFAT/POSIX constraints.
- **#12** — Painter M1/8GB compatibility gate; independent of Storage Tiering.

## 12. Decision

**Accepted:**

> Internal SSD is the control plane and hot core; Samsung T7 is the large warm working tier; cloud is an explicit cold archive tier; backup remains an independent recovery responsibility.

This is sufficient to close Issue #6. No real data movement is authorized by this decision.