# Submission types

The submission type decides which files sit in the version directory and who runs the verifier.

| Type | Files in the version directory | Who runs the verifier |
|---|---|---|
| Chart source | `src/` tree | CI |
| Tarball | `<chart>-<version>.tgz` | CI |
| Report only | `report.yaml` (+ `report.yaml.asc` if signed) | you |
| Report + chart | `report.yaml` plus `src/` or `.tgz` | CI re-runs with `-e has-readme` only |

`src/` and a tarball must never both be present — CI errors with
`Both chart source directory and tarball should not exist`. Web-catalog-only delivery **must**
be report-only, and the report must carry a package digest.

**If you supply a report, CI does not re-verify.** Your report's results *are* the verdict —
CI reads its `failed` count directly, and `chart-testing`, `images-are-certified` and the rest
are **not** re-run. Submit a chart with *no* report and CI instead runs the full verifier
against a live cluster and judges what it produces.

CI still ties the report to a real chart. With a chart attached, its `-e has-readme` run
recomputes the chart digest and compares it against your report (`Digest is not matching`
otherwise) — that digest check is what attaching the chart buys you. Report-only, CI instead
GETs the report's own `chart-uri` and sha256s the bytes against `digests.package`: a mismatch
fails the submission, while an unreachable URL only logs a warning and skips the comparison
(a malformed or schema-less URL *does* fail). Only web-catalog-only submissions skip this
entirely.

Chart name and version are checked on *every* submission, report-only included: they must
match the `charts/<category>/<org>/<chart>/<version>/` path, and the generated report is
brought into that comparison only when one exists.
