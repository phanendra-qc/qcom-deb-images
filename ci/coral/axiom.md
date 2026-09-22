# Coral / Axiom — GitHub Workflow Dependencies

> **Purpose**: This file is a heads-up reference for `qcom-deb-images`
> maintainers.  Before modifying any GitHub Actions workflow listed here,
> check whether it is a dependency of a Coral workflow — breaking it will
> block Coral pre-merge validation or nightly tests.

Coral project: **QLI Open Development**
Coral base URL: <https://qswat.qualcomm.com/coral/qli-open-development/workflows/>

---

## Pre-merge

**Coral workflow**: `Debian_Pre-Merge_Workflow`

| Property | Value |
|----------|-------|
| Coral URL | <https://qswat.qualcomm.com/coral/qli-open-development/workflows/Debian_Pre-Merge_Workflow> |
| Purpose | PR-level Axiom validation before merge |
| Coral Invoke | `pull_request` |
| Github workflow | `Build on PR` |
| Checkrun | `chekrun as build (trixie, default) / Build and upload debos recipes (trixie, default)` |
| Workflow ID | `159226426` |

> ⚠️ **Note**: `test-on-pr.yml` currently excludes `glymur-crd` from its
> `boards_exclude` list, so the `test (trixie) / Submit glymur-crd boot`
> check-run is **not produced** by the pre-merge workflow.  The Coral gate
> therefore tracks the debos build check-run instead (see table above).

### GitHub workflow dependency chain

```
pull_request
│
└─► build-on-pr.yml  (ID: 159226426)
    "Build on PR"
    https://github.com/qualcomm-linux/qcom-deb-images/actions/workflows/build-on-pr.yml
    │
    ├─ job: build  (matrix: suite × [trixie, forky])
    │   └─► debos.yml  (ID: 151938321)  "Build debos recipe"
    │       └─ job: build-debos
    │           ★ check-run: "Build and upload debos recipes (<suite>)"
    │           e.g.  "Build and upload debos recipes (trixie)"
    │                  ▲▲▲  CORAL GATE CHECK-RUN  ▲▲▲
    │           output: artifacts_url  ──────────────────────────────────────────────────┐
    │                                                                                     │
    └─ job: schema-check                                                                  │
        └─► lava-schema-check.yml  (ID: 163868026)  "Check LAVA templates"               │
            └─ job: schema-check                                                          │
                check-run: "schema-check"                                                 │
                                                                                          │
[after build-on-pr.yml completes — workflow_run trigger]                                  │
│                                                                                         │
└─► test-on-pr.yml  (ID: 281462353)                                                       │
    "Test PR build"                                                                       │
    │                                                                                     │
    ├─ job: retrieve-build-url  (downloads artifacts_url from build-on-pr run) ◄──────────┘
    │
    ├─ job: test  (suite: trixie)
    │   boards_exclude: ["glymur-crd", "monaco-evk", "qcs615-ride",
    │                    "qcs8300-ride", "lemans-evk"]
    │   └─► lava-test.yml  (ID: 193557765)  "Tests"
    │       │
    │       ├─ job: prepare-job-list
    │       ├─ job: submit-job  (matrix per ci/lava/ board template,
    │       │                    glymur-crd excluded — no Coral gate check-run)
    │       └─ job: publish-test-results
    │           check-run: "Publish Tests Results"
    │
    └─ job: publish-test-results  (PR comment with LAVA job URLs)
```

### Check-run Coral gates on

| Check-run name | Workflow | Job | Workflow ID |
|----------------|----------|-----|-------------|
| `Build and upload debos recipes (trixie)` | `debos.yml` | `build-debos` | `151938321` |

### Workflows that must not break

| Workflow file | Workflow name | Workflow ID | Role |
|---------------|--------------|-------------|------|
| `build-on-pr.yml` | Build on PR | `159226426` | Entry point — triggered on every PR |
| `debos.yml` | Build debos recipe | `151938321` | Builds rootfs + images; produces `artifacts_url` and the Coral gate check-run |
| `lava-schema-check.yml` | Check LAVA templates | `163868026` | Validates `ci/lava/` YAML before test submission |
| `test-on-pr.yml` | Test PR build | `281462353` | `workflow_run` bridge; submits LAVA jobs and posts check-runs back to the PR |
| `lava-test.yml` | Tests | `193557765` | Submits LAVA jobs; `glymur-crd` currently excluded from pre-merge runs |

---

## Post-merge

**Coral workflow**: `Debian_CRM_Nightly_Test_Hyd`

| Property | Value |
|----------|-------|
| Coral URL | <https://qswat.qualcomm.com/coral/qli-open-development/workflows/Debian_CRM_Nightly_Test_Hyd> |
| Purpose | Nightly Axiom build and hardware tests after merge to `main` |
| Coral Invoke | `push` to `main` |
| Github workflow | `Build on push to branch` |
| Checkrun | `Build and upload debos recipes (trixie)` |
| Workflow ID | `build-on-push.yml` |

> ⚠️ **Note**: `build-on-push.yml` also excludes `glymur-crd` from its
> `boards_exclude` list, so the `test (trixie) / Submit glymur-crd boot`
> check-run is **not produced** by the post-merge workflow either.

### GitHub workflow dependency chain

```
push to main
│
└─► build-on-push.yml
    "Build on push to branch"
    https://github.com/qualcomm-linux/qcom-deb-images/actions/workflows/build-on-push.yml
    Guard: branches: [main]
    │
    ├─ job: build-daily  (matrix: suite × [trixie, forky])
    │   └─► debos.yml  (ID: 151938321)  "Build debos recipe"
    │       └─ job: build-debos
    │           ★ check-run: "Build and upload debos recipes (<suite>)"
    │           e.g.  "Build and upload debos recipes (trixie)"
    │                  ▲▲▲  CORAL GATE CHECK-RUN  ▲▲▲
    │           output: artifacts_url  ──────────────────────────────────────┐
    │                                                                         │
    ├─ job: schema-check                                                      │
    │   └─► lava-schema-check.yml  "Check LAVA templates"                    │
    │                                                                         │
    └─ job: test  (suite: trixie, needs: build-daily + schema-check)         │
        boards_exclude: ["glymur-crd", "monaco-evk", "qcs615-ride",  ◄───────┘
                         "qcs8300-ride", "lemans-evk"]
        └─► lava-test.yml  (ID: 193557765)  "Tests"
            │
            ├─ job: prepare-job-list
            ├─ job: submit-job  (matrix per ci/lava/ board template,
            │                    glymur-crd excluded — no Coral gate check-run)
            └─ job: publish-test-results
                check-run: "Publish Tests Results"
```

### Check-run Coral gates on

| Check-run name | Workflow | Job | Workflow ID |
|----------------|----------|-----|-------------|
| `Build and upload debos recipes (trixie)` | `debos.yml` | `build-debos` | `151938321` |

### Workflows that must not break

| Workflow file | Workflow name | Role |
|---------------|--------------|------|
| `build-on-push.yml` | Build on push to branch | Entry point — triggered on every push to `main` |
| `debos.yml` | Build debos recipe | Builds rootfs + images; produces `artifacts_url` and the Coral gate check-run |
| `lava-test.yml` | Tests | Submits LAVA jobs; `glymur-crd` currently excluded from post-merge runs |

---

## Shared dependency: `debos.yml` and `lava-test.yml`

Both Coral workflows ultimately depend on the same components:

| Component | What it does | Impact if broken |
|-----------|-------------|-----------------|
| `.github/workflows/debos.yml` (ID: `151938321`) | Builds rootfs, disk images, flash artifacts; produces `artifacts_url`; job name is the Coral gate check-run | Coral gate check-run never appears → both workflows blocked |
| `.github/workflows/lava-test.yml` (ID: `193557765`) | Scans `ci/lava/`, builds board matrix, submits LAVA jobs, publishes check-runs | LAVA tests never run; `glymur-crd` currently excluded from both callers |
| `ci/lava/glymur-crd/boot.yaml` | LAVA job definition for the `glymur-crd` board | `glymur-crd` dropped from matrix if re-included; currently excluded |

**Key `debos.yml` behaviours to preserve**:
- Job name format: `Build and upload debos recipes (${{ inputs.suite }})` — Coral
  matches on the exact string `Build and upload debos recipes (trixie)`.
- `artifacts_url` output — consumed by `test-on-pr.yml` and `build-on-push.yml`
  to pass the build URL to `lava-test.yml`.

**Key `lava-test.yml` behaviours to preserve**:
- Check-run name format: `Submit <board> <test-name>` — if `glymur-crd` is
  re-added to callers, Coral would match on
  `test (trixie) / Submit glymur-crd boot`.
- `result_file_name` pattern: `test-results-<suite>-<slug>` — used by
  `publish-test-results` to download artifacts; changing it breaks result
  publishing.
- `boards_exclude` in both `build-on-push.yml` and `test-on-pr.yml` currently
  includes `glymur-crd`; removing it would re-enable the board for LAVA tests.

---

## Quick-reference: Coral gate check-run

Both Coral workflows gate on:

```
Build and upload debos recipes (trixie)
```

Produced by: `debos.yml` → job `build-debos`, suite `trixie`.

> If `glymur-crd` is re-enabled in both callers' `boards_exclude`, the Coral
> gate check-run would shift to:
> ```
> test (trixie) / Submit glymur-crd boot
> ```
> Produced by: `lava-test.yml` → job `submit-job` → matrix entry for
> `ci/lava/glymur-crd/boot.yaml`, suite `trixie`.
