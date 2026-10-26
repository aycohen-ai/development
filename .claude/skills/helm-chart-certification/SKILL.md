---
name: helm-chart-certification
description: Take a Helm chart from source to a submitted Red Hat certification PR against openshift-helm-charts/charts. Use whenever the user mentions chart-verifier, openshift-helm-charts, charts.openshift.io, Helm chart certification, an OWNERS file for a chart, a verify-report / report.yaml, chart provenance or .prov files, supportedOpenShiftVersions, webCatalogOnly, or a certification PR failing on a Helm chart. This is the Helm path of pipeline steps 5b and 7b — not the container or operator path.
---

# Helm Chart Certification

Pipeline steps **5b** (verify) and **7b** (submit). Drives a chart through `chart-verifier`,
packaging, and a PR to `openshift-helm-charts/charts`.

**Use this path when** the deliverable is a Helm chart. Use the container path when it is an
image, the operator path when it is a bundle/CSV. A chart whose images are not yet certified is
blocked on the container path first.

Related skills — defer to them, do not restate: `pyxis-product-catalog` (image certification
lookups), `pipeline-monitoring` (watching CI), `certification-pr-submission` (fork/branch/PR
mechanics).

---

## 1. Prerequisites

| Input | Required | Note |
|---|---|---|
| `Chart.yaml`, `values.yaml`, `values.schema.json`, `README.md`, `templates/tests/` | yes | each backs a mandatory check |
| `kubeVersion` in `Chart.yaml` | yes | drives `supportedOpenShiftVersions` |
| All referenced images certified and published | yes | check via `pyxis-product-catalog` |
| Red Hat Connect Helm chart project | partners | creates the `OWNERS` file — § 7 |
| Fork of `openshift-helm-charts/charts` | yes | |
| OpenShift cluster + `KUBECONFIG` | for `chart-testing` | not needed if CI runs the verifier for you |
| GPG key pair | only if signing | § 5 |

---

## 2. Pick the submission type

| Type | Files in the version directory | Who runs the verifier |
|---|---|---|
| Chart source | `src/` tree | CI |
| Tarball | `<chart>-<version>.tgz` | CI |
| Report only | `report.yaml` (+ `report.yaml.asc` if signed) | you |
| Report + chart | `report.yaml` plus `src/` or `.tgz` | CI re-runs with `-e has-readme` only |

`src/` and a tarball must never both be present — CI errors with
`Both chart source directory and tarball should not exist`. Web-catalog-only delivery **must**
be report-only, and the report must carry a package digest.

---

## 3. Run chart-verifier

CI installs the verifier as a binary via `redhat-actions/openshift-tools-installer`, tracking
**`latest`** for the production charts repo (no pinned digest). Locally, run the container:

```bash
podman run --rm -i \
  -e KUBECONFIG=/.kube/config \
  -v "${HOME}/.kube":/.kube:z \
  -v "$(pwd)/chartverifier":/app/chartverifier:z \
  quay.io/redhat-certification/chart-verifier:latest \
  verify -w \
  --set profile.vendorType=partner \
  --set profile.version=v1.1 \
  <chart-uri>
```

Writes `./chartverifier/report.yaml`, overwriting any existing file; without `-w` the report
goes to stdout.

| Flag | Effect |
|---|---|
| `-e, --enable <names>` | run only these checks |
| `-x, --disable <names>` | skip these checks |
| `-w` | write `report.yaml` under `/app/chartverifier` |
| `-o, --output` | `default`, `json`, `yaml` |
| `-V, --openshift-version` | sets `certifiedOpenShiftVersions` |
| `-W, --web-catalog-only` | distribution is web catalog only |
| `--pgp-public-key <file>` | key for `signature-is-valid` |
| `--timeout` / `--helm-install-timeout` | default 30m / 5m |
| `-S`, `-F` | chart value overrides |
| `--set images-are-certified.kube-version=v1.29` | kube version for the image check |

`-e`/`-x` are for local triage. CI itself uses `-x` never, and `-e has-readme` only when you
already supplied a report. A submission report with mandatory checks missing fails with
`Missing mandatory check : v1.0/<name>`.

### Profiles

`profile.vendorType` ∈ `partner | redhat | community | default` (`default` = `partner`);
`profile.version` ∈ `v1.0 | v1.1 | v1.2 | v1.3`, **defaulting to 1.1** when the report omits it.
The report's vendor type must match the category segment of your path (`partners` → `partner`),
or CI exits with
`[ERROR] Report profile type 'X' doesn't match the vendor type in the directory path`.

---

## 4. Mandatory checks and reading the report

The mandatory set is owned by chart-verifier, not by the CI repo — CI only reads the aggregate
counts. Thirteen checks carry `type: Mandatory` in an accepted partner report:

| Check | Verifies |
|---|---|
| `v1.0/is-helm-v3` | chart is Helm v3 |
| `v1.0/has-readme` | `README.md` present |
| `v1.0/contains-test` | at least one test file |
| `v1.0/contains-values` | `values.yaml` present |
| `v1.0/contains-values-schema` | `values.schema.json` present |
| `v1.1/has-kubeversion` | `kubeVersion` present and a valid semver range |
| `v1.0/not-contains-crds` | no CRDs |
| `v1.0/not-contain-csi-objects` | no CSI objects |
| `v1.1/images-are-certified` | every referenced image is Red Hat–certified |
| `v1.0/helm-lint` | `helm lint` passes |
| `v1.0/chart-testing` | installs and tests the chart on a real OCP cluster |
| `v1.0/required-annotations-present` | `charts.openshift.io/name` annotation present |
| `v1.0/signature-is-valid` | signed chart matches the public key (`SKIPPED` when unsigned) |

`has-notes` (NOTES.txt) is optional in every profile. In the `community` profile only
`helm-lint` is mandatory.

**Gate:** CI fails the PR iff `failed > 0`. There is no minimum passed count and no percentage
threshold. Per category:

- `partners` → `sys.exit(1)` unless the PR carries the maintainer-only `force-publish` label.
- `redhat` → not a hard fail; the chart is demoted to `providerType: community`.
- `community` → **always** requires manual maintainer review, even at zero failures, unless
  `force-publish` is set.

**Report shape:** `apiversion: v1`, `kind: verify-report`, then `metadata.tool`
(`verifier-version`, `profile.VendorType`, `profile.version`, `reportDigest`, `chart-uri`,
`digests.chart`, `digests.package`, `digests.publicKey`, `lastCertifiedTimestamp`,
`testedOpenShiftVersion`, `supportedOpenShiftVersions`, `webCatalogOnly`), `metadata.chart`
(including `kubeversion`), `metadata.chart-overrides`, and `results[]` of
`{check, type, outcome, reason}`. CI rejects the report unless `kind` is `verify-report` and
`results`, `metadata`, `metadata.tool`, `metadata.chart` all exist.

**Required annotations** (profile ≥ v1.1) — a missing one is a hard exit:

```
charts.openshift.io/lastCertifiedTimestamp
charts.openshift.io/testedOpenShiftVersion
charts.openshift.io/supportedOpenShiftVersions
charts.openshift.io/digest
```

Profile v1.0 instead requires `lastCertifiedTimestamp`, `certifiedOpenShiftVersions`, `digest`.

**kubeVersion cross-checks**, applied only when `chart-testing` passed: the tested/certified OCP
version must be valid semver and `>= 4.1.0`; for v1.1+ it must fall inside the OCP range derived
from `kubeVersion`, and `supportedOpenShiftVersions` must be present and **string-equal** to
that derived range, else `Kube Version ... does not match supportedOpenShiftVersions`. The
range is computed by the external `get-ocp-range` tool (`github.com/opdev/getocprange`); the
repo holds no OCP↔kubeVersion table. A range-derivation failure is tolerated only for profile
v1.0.

Since chart-verifier 1.9.0 the report carries a content-derived sha — never hand-edit a report.

---

## 5. Package, version, and sign

```bash
helm package ./<chart-name>
```

- `version` in `Chart.yaml` must equal the version directory name and the version in
  `report.yaml`, and must be valid semver.
- The version must be new: absent from `index.yaml` and with no existing git tag. Published
  versions are immutable — ship a new version.
- A tarball must be named exactly `<chart-name>-<version>.tgz`, and must contain a single
  top-level directory named after the chart holding `Chart.yaml`, with nothing at the archive
  root.
- If the organization directory is `redhat`, the chart name must start with `redhat-`; no other
  organization may use that prefix.

Signing a chart (produces `.prov`):

```bash
helm package --sign --key '<key-name>' --keyring ~/.gnupg/secring.gpg ./<chart-name>
gpg --export -a '<key-name>' > public.key
```

A chart counts as signed when the PR carries both `.tgz` and the matching `.tgz.prov`. CI then
base64-decodes `publicPgpKey` from `OWNERS` and passes `--pgp-public-key` to the verifier, and
compares the sha256 of that key against `metadata.tool.digests.publicKey` in the report. A
mismatch fails with `PGP key in OWNERS file does not match with key digest in report.` Omitting
the key skips the check.

Signing a *report* (detached, ASCII armor) is **advisory only** — a bad signature prints a
`[WARNING]` and never fails the PR:

```bash
gpg --sign --armor --detach-sign --output report.yaml.asc report.yaml
```

The release attaches `<chart>-<version>.tgz`, `.tgz.prov`, the generated `.tgz.key`, and
`report.yaml`. No release is generated for web-catalog-only delivery.

---

## 6. Submit the PR

Path, matched as `charts/(partners|redhat|community)/<organization>/<chart>/<version>/`:

```
charts/partners/acme/awesome/0.1.0/src/Chart.yaml
charts/partners/acme/awesome/0.1.0/src/values.yaml
charts/partners/acme/awesome/0.1.0/src/values.schema.json
charts/partners/acme/awesome/0.1.0/src/README.md
charts/partners/acme/awesome/0.1.0/src/templates/tests/test-connection.yaml
charts/partners/acme/awesome/0.1.0/report.yaml          # optional
charts/partners/acme/awesome/0.1.0/awesome-0.1.0.tgz    # alternative to src/
charts/partners/acme/awesome/0.1.0/awesome-0.1.0.tgz.prov
```

`OWNERS` sits one level up, with **no version segment**, at
`charts/partners/acme/awesome/OWNERS`:

```yaml
chart:
  name: awesome
  shortDescription: A Helm chart for Awesomeness
publicPgpKey: null
providerDelivery: False
users:
  - githubUsername: <username-one>
vendor:
  label: acme
  name: ACME Inc.
```

- `chart.name` must equal the chart directory; `vendor.label` must equal the organization
  directory.
- The PR author must appear in `users[].githubUsername`, else
  `[ERROR] <user> is not allowed to submit the chart on behalf of <organization>`.
- `providerDelivery` (newer alias `web_catalog_only`) must agree with `webCatalogOnly` in the
  report.
- **Partners must never submit `OWNERS` themselves** — `redhat-mercury-bot` pushes it on behalf
  of Connect (§ 7). Community and Red Hat submitters open an `OWNERS`-only PR to `main` *first*,
  under `charts/community/<org>/<chart>/OWNERS` or `charts/redhat/redhat/<chart>/OWNERS`. Red Hat
  OWNERS must use label `redhat`, vendor name `Red Hat`, and a `redhat-` chart prefix.
- An `OWNERS` file mixed into a chart PR fails with
  `[ERROR] Send OWNERS file by itself in a separate PR.`
- **One chart per PR** — a second chart raises
  `[ERROR] A PR must contain only one chart...`. Any file outside the recognised set
  (report / source / tarball) is rejected as unknown.

### What CI runs (workflow `CI`, concurrency `cert-ci-<PR#>`)

| Job | Enforces |
|---|---|
| Setup CI | authorizes workflow-only changes, selects the verifier version |
| Extract and validate PR content | one chart, semver version, allowed file set, `redhat-` prefix, tarball naming and contents, webCatalogOnly coherence, version absent from `index.yaml`, release tag unused → adds the `content-ok` label |
| Run chart-verifier | logs into the cluster, creates per-PR namespace `charts-<PR#>`, runs the verifier when no report was supplied, derives the OCP range, runs the report review, uploads errors, tears the namespace down |
| Comment and merge PR | posts the PR comment, adds `authorized-request`, approves, `gh pr merge --auto --squash` |
| Release Chart | serialized on `release-index-update`; creates the GitHub release (skipped when web-catalog-only) and updates the index |

Release tag and index commit: `<organization>-<chart-name>-<version>`, e.g.
`acme-psql-service-0.1.1`. The index file is `index.yaml` on `gh-pages`, or
`unpublished-certified-charts.yaml` for web-catalog-only. The bot's comment carries a single
`/metadata {"vendor_label": "...", "chart_name": "..."}` line. Watch the run with
`pipeline-monitoring`.

---

## 7. Red Hat Connect portal — human hand-offs

No API equivalent exists in the CI repo. Stop, hand over, wait.

> **HUMAN HAND-OFF — create the product listing.** Partner Connect → *Certified technology
> portal*. Create a listing by product category, complete the company profile, accept the legal
> agreements, add at least one component. Hand over: chart name, vendor name, vendor label.
> Wait for: the listing to exist before any component work.

> **HUMAN HAND-OFF — create and configure the Helm chart component.** Partner Connect →
> component → *Repository information* and *Component details* tabs. Set the container registry
> namespace, the Helm chart repository location, access instructions, and the public PGP key if
> signing. Hand over: the ASCII-armored public key and the GitHub usernames to authorize.
> Wait for: `redhat-mercury-bot` to land the `OWNERS` file at
> `charts/partners/<org>/<chart>/OWNERS`. Do not proceed to § 6 until it exists.

> **HUMAN HAND-OFF — certify the referenced container images.** Every image in the chart needs
> its own certified, published container component before `images-are-certified` can pass. Hand
> over: the image list from the rendered templates. Wait for: *Published* status in the Red Hat
> Ecosystem Catalog; confirm with `pyxis-product-catalog`.

> **HUMAN HAND-OFF — publish.** Partner Connect → *My Work* → *Product Listings* → the listing →
> *Certification Projects* → *Attach Project*. Attach the chart project and the certified
> container project, both *Published*, then *Publish*. Hand over: the merged PR URL and the
> release tag. Wait for: the chart appearing in the Red Hat Ecosystem Catalog.

---

## 8. Common failures

| Symptom | Cause | Fix |
|---|---|---|
| `images-are-certified` FAIL | image not certified/published, or pulled by floating tag from an uncertified registry | certify via the container path; pin a published tag or digest; re-run verify |
| `has-kubeversion` FAIL | `kubeVersion` absent, or not a valid semver range under v1.1+ | add e.g. `kubeVersion: ">=1.25.0-0"`; re-run verify to regenerate `supportedOpenShiftVersions` |
| `[ERROR] Missing annotation in chart/report: charts.openshift.io/supportedOpenShiftVersions` | report generated under profile v1.0, or hand-edited | regenerate with `--set profile.version=v1.1`; never edit the report |
| `Kube Version ... does not match supportedOpenShiftVersions` | annotation not string-equal to the range derived from `kubeVersion` | regenerate the report; do not hand-write the annotation |
| `testedOpenShiftVersion` outside `supportedOpenShiftVersions`, or `< 4.1.0` | verifier ran on an out-of-range or unsupported cluster | widen `kubeVersion` or re-run on an in-range cluster |
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
| Community PR green but not merged | community always needs maintainer review | request review; only a maintainer can apply `force-publish` |

---

## 9. Verification checklist

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
(§ 7) is a separate human hand-off, outside this criterion.

---

## 10. Reference

- Checks and profiles: `redhat-certification/chart-verifier`, `docs/helm-chart-checks.md`
- Submission guide and the full error catalogue: `docs/README.md` in this repo; also
  `docs/pull-request.md` (PR contract) and `docs/workflow-build.md` (per-category behaviour)
- Red Hat Software Certification Workflow Guide, "Working with Helm charts"
- Gating logic: `scripts/src/chartprreview/chartprreview.py`,
  `scripts/src/report/verifier_report.py`, `scripts/src/submission/submission.py`,
  `scripts/src/owners/owners_file.py`, `scripts/src/signedchart/signedchart.py`,
  `scripts/src/reporegex/matchers.py`, `.github/workflows/build.yml`
