# Running chart-verifier and reading the report

## How CI runs the verifier

CI installs the verifier as a binary via `redhat-actions/openshift-tools-installer`, tracking
**`latest`** for the production charts repo (no pinned digest). CI runs it with `fail: false`,
so failed checks do not fail that step — the pass/fail gate is the later *Check Report* step.
A red *Run chart-verifier* step means the verifier itself errored, not that a check failed.

## Running it locally

Run the container:

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

## Profiles

`profile.vendorType` ∈ `partner | redhat | community | default` (`default` = `partner`);
`profile.version` ∈ `v1.0 | v1.1 | v1.2 | v1.3`, **defaulting to 1.1** when the report omits it.
The report's vendor type must match the category segment of your path (`partners` → `partner`),
or CI exits with
`[ERROR] Report profile type 'X' doesn't match the vendor type in the directory path`.

## Mandatory checks

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

## Report shape

`apiversion: v1`, `kind: verify-report`, then `metadata.tool`
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

## kubeVersion cross-checks

Applied only when `chart-testing` passed: the tested/certified OCP version must be valid semver
and `>= 4.1.0`; for v1.1+ it must fall inside the OCP range derived from `kubeVersion`, and
`supportedOpenShiftVersions` must be present and **string-equal** to that derived range, else
`Kube Version ... does not match supportedOpenShiftVersions`. The range is computed by the
external `get-ocp-range` tool (`github.com/opdev/getocprange`); the repo holds no
OCP↔kubeVersion table. A range-derivation failure is tolerated only for profile v1.0.

Since chart-verifier 1.9.0 the report carries a content-derived sha — never hand-edit a report.
