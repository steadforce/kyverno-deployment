# Kyverno Deployment

This repository is an umbrella chart that packages and configures the upstream [Kyverno][kyverno] Helm chart for
our clusters, together with a set of cluster-wide policies.

> [!WARNING]
> Never install the content of this repository on a cluster manually. Deployment is done exclusively by ArgoCD.

## Overview

- **Umbrella chart**: [`Chart.yaml`](Chart.yaml) declares `kyverno` from `https://kyverno.github.io/kyverno/` as
  its only dependency, and [`Chart.lock`](Chart.lock) pins the exact resolved version and digest. The matching
  subchart archive is committed in [`charts/`](charts).
- **Cluster policies**: [`templates/`](templates) adds policies and supporting resources that are not part of the
  upstream chart (see [Cluster Policies](#cluster-policies)).
- **Separated override values**: subchart overrides live in their own value file, apart from the umbrella chart's
  own values (see [Value Files](#value-files)).
- **Unit tests**: the helm-unittest suites in [`tests/`](tests) cover the rendered manifests of both the root
  templates and the subchart.

## Prerequisites

Commands in this document run from the repository root, in one of two ways:

- From the `SteadOps-Steadies-K8s-Workplace` workbench, which already provides `helm`, `yq`, `kubectl`,
  `hetzner-k3s`, and `act`. Tool commands run directly from the workbench shell.
- From any machine with Docker installed, using the containerized examples.

## Repository Layout

| Path | Purpose |
| --- | --- |
| [`Chart.yaml`](Chart.yaml) | Umbrella chart metadata and the `kyverno` dependency. |
| [`Chart.lock`](Chart.lock) | Committed lock file pinning the resolved dependency version and digest. |
| [`charts/`](charts) | Committed subchart archive matching `Chart.lock`. |
| [`values.yaml`](values.yaml) | Umbrella chart defaults, always applied. |
| `values-*.yaml` | Subchart overrides and environment-specific overrides (see [Value Files](#value-files)). |
| [`templates/`](templates) | Cluster policies and resources added on top of the `kyverno` subchart. |
| [`tests/`](tests) | helm-unittest suites. |
| [`.github/workflows/`](.github/workflows) | CI workflows calling reusable workflows. |
| [`renovate.json`](renovate.json) | Renovate dependency update rules. |

`_local/`, `tests/__snapshot__/`, and `test-output.xml` are generated locally and gitignored.

## Cluster Policies

The following resources are added on top of the upstream chart:

- `disallow-latest-tag` - a `ClusterPolicy` that requires an image tag and denies pods using the mutable
  `:latest` tag. Its `validationFailureAction` is set by `policies.disallowLatestTag.validationFailureAction`.
  It renders only when the `kyverno.io/v1` API is available.
- `add-ttl-jobs` - a `ClusterPolicy` that adds `ttlSecondsAfterFinished: 900` to `Job` resources without an owner
  reference, unless already set, so they get cleaned up automatically.
- `clean-old-failed-jobs` - a `ClusterCleanupPolicy` that removes failed jobs every five minutes.
- `kyverno:generate-issuer` - a `ClusterRole` granting Kyverno permission to manage cert-manager `Issuer`
  resources. It renders only when both `rbac.authorization.k8s.io/v1` and `cert-manager.io/v1` are available.
- The release namespace, labeled `name: <namespace>`.

## Value Files

| File | Scope | Purpose |
| --- | --- | --- |
| [`values.yaml`](values.yaml) | All environments | Policy defaults. |
| [`values-subchart-overrides.yaml`](values-subchart-overrides.yaml) | All environments | Subchart overrides. |
| [`values-local.yaml`](values-local.yaml) | Local clusters | Single replicas, near-zero resources, audit only. |
| [`values-production.yaml`](values-production.yaml) | Production clusters | Audit-only `disallow-latest-tag`. |
| [`values-sf-k8s04-dev.yaml`](values-sf-k8s04-dev.yaml) | `sf-k8s04-dev` | Admission controller memory request. |

`values.yaml` sets `policies.disallowLatestTag.validationFailureAction` to `enforce`, and
`policies.requireRequestsAndLimits.validationFailureAction` to `audit`, which no template currently uses.
`values-local.yaml` and `values-production.yaml` switch `disallow-latest-tag` to `audit`.

`values-subchart-overrides.yaml` sets, under the top-level `kyverno:` key, the replicas, resources, priority
class, and additional cluster role permissions of the controllers, the resources of the cleanup jobs, the
`ServerSideApply=true` ArgoCD sync option on the CRDs, the enabled features (policy exceptions in all namespaces,
no admission reports), a network policy, and the `bitnamilegacy/kubectl` image for the cleanup hooks. The
webhooks cleanup hook is disabled because ArgoCD does not support pre-delete Helm hooks.

> [!NOTE]
> Subchart overrides live in their own file because Helm does not allow disabling the use of `values.yaml`.
> Keeping them separate lets the unit tests catch incompatible changes in the subchart's values, for example
> whether the umbrella chart and the subchart reference the same image registry and repository.

## Setup

`Chart.lock` and the matching subchart archive in `charts/` are committed, so a fresh clone can be rendered and
tested right away. After a pull that changes `Chart.lock`, or when `charts/` got out of sync, rebuild it with
`helm dependency build`, which downloads exactly the pinned subchart. The dependency repositories are derived
from `Chart.yaml`, the same way the pipeline does it, because `helm dependency build` only resolves registered
repositories.

From the workbench:

```shell
 yq 'explode(.) | .dependencies[] | select(.repository == "http*") | .name + " " + .repository' Chart.yaml |
   while read -r name repo; do helm repo add --force-update "$name" "$repo"; done
 helm dependency build .
```

Containerized:

```shell
 docker run \
   -e HOME=/tmp \
   --entrypoint sh \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm -c '
     yq "explode(.) | .dependencies[] | select(.repository == \"http*\") | .name + \" \" + .repository" Chart.yaml |
       while read -r name repo; do helm repo add --force-update "$name" "$repo"; done &&
     helm dependency build .
   '
```

## Rendering

This renders the chart for a local cluster into `_local/local/`. The `-a` flags declare the APIs that the
policies and the subchart check for, since no cluster is queried while rendering. For another environment,
replace `values-local.yaml` with its value file, or drop it for the defaults.

From the workbench:

```shell
 helm template kyverno . \
   -a batch/v1/CronJob \
   -a cert-manager.io/v1 \
   -a kyverno.io/v1 \
   -f values-subchart-overrides.yaml \
   -f values-local.yaml \
   --include-crds \
   -n kyverno \
   --output-dir _local/local \
   --skip-tests
```

Containerized:

```shell
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm template kyverno . \
   -a batch/v1/CronJob \
   -a cert-manager.io/v1 \
   -a kyverno.io/v1 \
   -f values-subchart-overrides.yaml \
   -f values-local.yaml \
   --include-crds \
   -n kyverno \
   --output-dir _local/local \
   --skip-tests
```

> [!NOTE]
> `-f` flags are applied in order, with later files overriding earlier ones. Keep
> `values-subchart-overrides.yaml` first and the environment-specific file last.

## Testing

The suites in [`tests/`](tests) assert, per controller (admission, background, cleanup, and reports), the
container resources for local and all other clusters, the additional cluster role permissions, the policy
exceptions feature, and the logging verbosity and format. They also assert that `disallow-latest-tag` is enforced
by default and on `sf-k8s04-dev`, and only audited on local and production clusters.

```shell
 docker run \
   -e HELM_CACHE_HOME=/tmp/helm/.config \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   helmunittest/helm-unittest .
```

helm-unittest writes XUnit by default. To get a JUnit report in `test-output.xml`, as the pipeline does, add
`-t JUnit -o test-output.xml` before the chart path. Inside the workbench, run `helm unittest .` directly.

### Run GitHub Workflows Locally

From the workbench, run `act` in the repository root. On first execution you are asked which flavor of the `act`
image to use; the default `medium` is a good starting point. Under `act`, the unit test workflow skips publishing
test results and sending notifications.

## CI/CD

All workflows in [`.github/workflows/`](.github/workflows) call reusable workflows from
[`steadforce/steadops-workflows`][steadops-workflows], pinned to `v4.2.0`:

| Workflow | Trigger | Purpose |
| --- | --- | --- |
| [`helm-unittest.yaml`](.github/workflows/helm-unittest.yaml) | Every push | Unit tests, lint, notifications. |
| [`trufflehog.yaml`](.github/workflows/trufflehog.yaml) | Push/PR to `main`, manual | Scans commits for secrets. |

- **Unit tests** register the chart's dependency repositories and run `helm dependency build` against
  `Chart.lock` (falling back to `helm dependency update` with a warning when the lock file is missing). They then
  run `helm unittest` including subchart tests, publish the JUnit results, and run `helm lint`.
- **Notifications**: on `renovate/` branches, the unit test result is posted to MS Teams. Successes go to the
  channel of the `STEADOPS_HELM_RENOVATION_MS_TEAMS_WEBHOOK` secret, failures to the separate error channel of the
  `STEADOPS_HELM_RENOVATION_ERROR_MS_TEAMS_WEBHOOK` secret. When the error secret is not set, failures go to the
  regular channel; when neither is set, no notification is sent.
- **Trufflehog** scans the pushed or pull request commit range and fails when the scan finds secrets.

## Dependency Updates

Dependency updates are managed by [Renovate][renovate] (see [`renovate.json`](renovate.json)):

- All GitHub Actions updates, including major ones, are merged automatically once checks pass (squash merge).
- All other updates, including every `kyverno` subchart update, are opened as pull requests for manual review.
- Renovate also updates the subchart archive in `charts/` (`helmUpdateSubChartArchives`).

To change the subchart version manually, edit the dependency version in `Chart.yaml`, then regenerate
`Chart.lock` and the archive in `charts/`:

```shell
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm dependency update .
```

Inside the workbench, run `helm dependency update .` directly. It replaces the old archive in `charts/` with the
new one. Commit both changes together with `Chart.yaml` and the new `Chart.lock`, otherwise the unit tests fail
because `Chart.lock` and `Chart.yaml` disagree. See the [Helm docs][helm-dependencies] for details.

[helm-dependencies]: https://helm.sh/docs/topics/charts/#chart-dependencies
[kyverno]: https://kyverno.io
[renovate]: https://docs.renovatebot.com
[steadops-workflows]: https://github.com/steadforce/steadops-workflows
