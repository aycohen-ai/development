# Packaging, versioning, and signing

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

## Signing a chart (produces `.prov`)

```bash
helm package --sign --key '<key-name>' --keyring ~/.gnupg/secring.gpg ./<chart-name>
gpg --export -a '<key-name>' > public.key
```

A chart counts as signed when the PR carries both `.tgz` and the matching `.tgz.prov`. CI then
base64-decodes `publicPgpKey` from `OWNERS` and passes `--pgp-public-key` to the verifier, and
compares the sha256 of that key against `metadata.tool.digests.publicKey` in the report. A
mismatch fails with `PGP key in OWNERS file does not match with key digest in report.`
Omitting the key skips the check.

## Signing a report

Detached, ASCII armor — **advisory only**. A bad signature prints a `[WARNING]` and never
fails the PR:

```bash
gpg --sign --armor --detach-sign --output report.yaml.asc report.yaml
```

## What the release attaches

The release attaches `<chart>-<version>.tgz`, `.tgz.prov`, the generated `.tgz.key`, and
`report.yaml`. No release is generated for web-catalog-only delivery.
