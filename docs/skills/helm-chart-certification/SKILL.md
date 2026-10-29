---
name: helm-chart-certification
description: Take a Helm chart from source to a submitted Red Hat certification PR against openshift-helm-charts/charts. Use whenever the user mentions chart-verifier, openshift-helm-charts, charts.openshift.io, Helm chart certification, an OWNERS file for a chart, a verify-report / report.yaml, chart provenance or .prov files, supportedOpenShiftVersions, webCatalogOnly, or a certification PR failing on a Helm chart. This is the Helm path of pipeline steps 5b and 7b — not the container or operator path.
license: Apache-2.0
---

# Helm Chart Certification

Pipeline steps **5b** (verify) and **7b** (submit). Drives a chart through `chart-verifier`,
packaging, and a PR to `openshift-helm-charts/charts`.

**Use this path when** the deliverable is a Helm chart. Use the container path when it is an
image, the operator path when it is a bundle/CSV. A chart whose images are not yet certified is
blocked on the container path first.

## Safety rules

- Never print, copy, or transmit kubeconfig contents, bearer tokens, registry credentials, or
  private keys. Signing uses the user's own GPG keyring; never generate or transmit key material.
- Never hand-edit a `report.yaml` — since chart-verifier 1.9.0 it carries a content-derived sha,
  and CI rejects an edited report.
- Chart-related and non-chart changes must never share a PR. One chart version per PR.
- Pushing a branch and opening the PR are outward-facing actions: confirm with the user first.

## Procedure

### 1. Confirm prerequisites

| Input | Required | Note |
|---|---|---|
| `Chart.yaml`, `values.yaml`, `values.schema.json`, `README.md`, `templates/tests/` | yes | each backs a mandatory check |
| `kubeVersion` in `Chart.yaml` | yes | drives `supportedOpenShiftVersions` |
| All referenced images certified and published | yes | check via `pyxis-product-catalog` |
| Red Hat Connect Helm chart project | partners | creates the `OWNERS` file — see [connect-handoffs.md](references/connect-handoffs.md) |
| Fork of `openshift-helm-charts/charts` | yes | |
| OpenShift cluster + `KUBECONFIG` | for `chart-testing` | not needed if CI runs the verifier for you |
| GPG key pair | only if signing | see [packaging-and-signing.md](references/packaging-and-signing.md) |

### 2. Pick the submission type

Chart source (`src/`), tarball, report only, or report + chart — the choice decides who runs the
verifier and which digest checks apply. See [submission-types.md](references/submission-types.md).

### 3. Run chart-verifier and read the report

Run the verifier (locally or via CI), confirm the profile matches the submission path, and check
that `failed` is `0`. Mandatory checks, report shape, required annotations, and the
kubeVersion cross-checks are covered in [verify-and-report.md](references/verify-and-report.md).

### 4. Package, version, and sign

`helm package`, semver and immutability rules, `.prov` and report signatures:
[packaging-and-signing.md](references/packaging-and-signing.md).

### 5. Submit the PR

Directory layout, the `OWNERS` file contract, and what CI runs on the PR:
[submit-pr.md](references/submit-pr.md).

### 6. Hand off portal work

Product listing, Helm component, image certification, and publishing are human steps in the Red
Hat Connect portal — no API equivalent exists. Stop, hand over, wait:
[connect-handoffs.md](references/connect-handoffs.md).

### 7. Troubleshoot failures

Symptom → cause → fix for every documented CI error, including the version-0-failures nuance
per category: [troubleshooting.md](references/troubleshooting.md).

## Verification checklist

- [ ] `version` identical in `Chart.yaml`, directory name, and `report.yaml`; valid semver; not already released
- [ ] `kubeVersion` set and a valid semver range
- [ ] every referenced image certified and published
- [ ] verify run with `profile.vendorType` matching the path category, and `failed: 0`
- [ ] all four required annotations present; `supportedOpenShiftVersions` matches the derived range
- [ ] report unedited since generation
- [ ] `OWNERS` exists one level above the version directory; chart name, vendor label, and author all match; not included in this PR
- [ ] exactly one of `src/` or a correctly named `.tgz`; `.prov` present if signed; `publicPgpKey` matches
- [ ] one chart, and nothing outside its version directory

**Done when** the PR carries `content-ok` and `authorized-request`, CI is green, the PR is
merged, the `<organization>-<chart>-<version>` release exists, and the chart appears in
`index.yaml` (or `unpublished-certified-charts.yaml` for web-catalog-only). Portal publishing
is a separate human hand-off, outside this criterion.

## Reference

- Checks and profiles: `redhat-certification/chart-verifier`, `docs/helm-chart-checks.md`
- Submission guide and the full error catalogue: `docs/README.md` in this repo; also
  `docs/pull-request.md` (PR contract) and `docs/workflow-build.md` (per-category behaviour)
- Gating logic: `scripts/src/chartprreview/chartprreview.py`,
  `scripts/src/report/verifier_report.py`, `scripts/src/submission/submission.py`,
  `scripts/src/owners/owners_file.py`, `scripts/src/signedchart/signedchart.py`,
  `scripts/src/reporegex/matchers.py`, `.github/workflows/build.yml`
