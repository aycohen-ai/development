# Troubleshooting PR failures

After submitting, keep the PR open and watch the checks. The first part of Chart Certification
(PR Content Check) fails if the pull request contains changes not related to any chart, the
chart version is already released, or provider control delivery settings are inconsistent. The
second part (Verify PR) performs the checks below. The third part makes the release and updates
`index.yaml`.

| Symptom | Cause | Fix |
|---|---|---|
| `images-are-certified` FAIL | image not certified/published, or pulled by floating tag from an uncertified registry | certify via the container path; pin a published tag or digest; re-run verify |
| `has-kubeversion` FAIL | `kubeVersion` absent, or not a valid semver range under v1.1+ | add e.g. `kubeVersion: ">=1.25.0-0"`; re-run verify to regenerate `supportedOpenShiftVersions` |
| `[ERROR] Missing annotation in chart/report: charts.openshift.io/supportedOpenShiftVersions` | report generated under profile v1.0, or hand-edited | regenerate with `--set profile.version=v1.1`; never edit the report |
| `Kube Version ... does not match supportedOpenShiftVersions` | annotation not string-equal to the range derived from `kubeVersion` | regenerate the report; do not hand-write the annotation |
| `Tested OpenShift version X not within specified kube-versions` / `... not within supported versions` | verifier ran outside the OCP range derived from `kubeVersion` | widen `kubeVersion` or re-run on an in-range cluster |
| `charts.openshift.io/testedOpenShiftVersion X is not a supported OpenShift version.` | verifier ran on OCP older than 4.1.0, the minimum with Helm support | re-run on OCP >= 4.1.0 |
| Report rejected / digest mismatch | report edited after generation (content sha since 1.9.0) | regenerate |
| Verifier version skew — a check unknown locally is required by CI | CI tracks `latest`; local image is older | pull `:latest` and re-run before pushing; re-check after any verifier release |
| `[ERROR] Report profile type 'X' doesn't match the vendor type in the directory path` | `profile.vendorType` ≠ the path category | re-run verify with the matching vendor type |
| OWNERS mismatch on chart or vendor name | `chart.name` ≠ chart directory, or `vendor.label` ≠ organization directory | partners: fix through Connect, never hand-edit; others: correct the `OWNERS` PR or move the path |
| `<user> is not allowed to submit the chart on behalf of <org>` | PR author missing from `users[]` | add via Connect (partners) or an `OWNERS`-only PR |
| `[ERROR] Send OWNERS file by itself in a separate PR.` | `OWNERS` bundled with chart files | split into two PRs |
| `[ERROR] A PR must contain only one chart...` | two charts in one PR | one chart version directory per PR |
| `PGP key in OWNERS file does not match with key digest in report.` | stale `publicPgpKey`, or chart signed with a different key | re-export, update `OWNERS`, regenerate the report with `--pgp-public-key` |
| Version rejected as already released | version present in `index.yaml` or tag exists | bump `version`, rename the directory, regenerate the report |
| `Both chart source directory and tarball should not exist` | `src/` and `.tgz` both submitted | keep one |
| `[ERROR] OWNERS file should never be set directly by partners. See certification docs.` | a partner PR touches `OWNERS` at all | drop it; `redhat-mercury-bot` pushes it on behalf of Connect |
| `[ERROR] the tgz file is named incorrectly. Expected: X. Got: Y` | tarball not named `<chart>-<version>.tgz` | rename; it must match `Chart.yaml` and the version directory |
| `[ERROR] Incorrect tarball content: found a file outside the <chart> directory` | archive has files at its root, or the wrong top-level directory | repackage with `helm package`; exactly one top-level directory holding `Chart.yaml` |
| `[ERROR] One or more files included in the pull request are not part of the chart` | a file outside the recognised report / source / tarball set | remove it; unrelated changes need their own PR |
| `[ERROR] The web catalog distribution method requires a package digest in the report.` | web-catalog-only report carries no package digest | regenerate with `-W` against the packaged chart, not the source tree |
| `[ERROR] Digest is not matching: <submitted>, <generated>` | report submitted alongside a chart it does not describe | regenerate the report from the exact chart in the PR |
| Community PR green but not merged | community always needs maintainer review | request review; only a maintainer can apply `force-publish` |
