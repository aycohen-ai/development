# Submitting the PR

## Directory layout

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

## The OWNERS file

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
  of Connect. Community and Red Hat submitters open an `OWNERS`-only PR to `main` *first*,
  under `charts/community/<org>/<chart>/OWNERS` or `charts/redhat/redhat/<chart>/OWNERS`. Red Hat
  OWNERS must use label `redhat`, vendor name `Red Hat`, and a `redhat-` chart prefix.
- An `OWNERS` file mixed into a chart PR fails with
  `[ERROR] Send OWNERS file by itself in a separate PR.`

## PR content rules

- **One chart per PR** — a second chart raises
  `[ERROR] A PR must contain only one chart...`. Any file outside the recognised set
  (report / source / tarball) is rejected as unknown.

## What CI runs (workflow `CI`, concurrency `cert-ci-<PR#>`)

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

## Post-submission review

1. Watch the PR for bot comments; errors route to [troubleshooting.md](troubleshooting.md).
2. On success, verify the `gh-pages` index commit `<org>-<chart>-<version> index.yaml (#<PR#>)`
   reflects your chart.
3. A GitHub release tagged `<org>-<chart>-<version>` appears with the chart and report.
4. The chart becomes installable from https://charts.openshift.io .
