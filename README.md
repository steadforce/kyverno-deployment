# Kyverno Deployment

Helm umbrella chart that deploys [Kyverno](https://github.com/kyverno/kyverno/) together with a set of
cluster-wide policies, packaged as manifests for our clusters.

> [!WARNING]
> Never install the content of this repository on a cluster manually. Deployment is done exclusively by ArgoCD.

## Overview

This chart wraps the upstream `kyverno` chart as a dependency and adds:

- Root-level `templates/` with cluster policies and supporting resources that are not part of the upstream chart.
- Root-level `values*.yaml` files that customize both the added policies and the `kyverno` subchart per environment.
- A Helm unittest suite under `tests/` that covers the rendered manifests of both the root templates and the
  subchart.

## Repository Structure

| Path                              | Description                                                             |
| ---------------------------------- | ------------------------------------------------------------------------ |
| `Chart.yaml`                       | Chart metadata and the `kyverno` subchart dependency declaration.        |
| `charts/`                          | Downloaded subchart archive, managed by `helm dependency update`.        |
| `templates/`                       | Cluster policies and resources added on top of the `kyverno` subchart.   |
| `tests/`                           | Helm unittest suites and generated snapshots (gitignored).               |
| `values.yaml`                      | Default values, always applied.                                          |
| `values-local.yaml`                | Overrides for local development clusters.                               |
| `values-production.yaml`           | Overrides for production clusters.                                      |
| `values-sf-k8s04-dev.yaml`         | Overrides for the `sf-k8s04-dev` cluster.                                |
| `values-subchart-overrides.yaml`   | Values forwarded to the `kyverno` subchart (see [Testing](#testing)).    |

## Cluster Policies

The following `ClusterPolicy` and `ClusterCleanupPolicy` resources are added on top of the upstream chart:

- `disallow-latest-tag` - denies pods using a mutable `:latest` image tag. Its `validationFailureAction` is
  driven by `policies.disallowLatestTag.validationFailureAction` in the root value files.
- `add-ttl-jobs` - mutates unowned `Job` resources to add `ttlSecondsAfterFinished` so they get cleaned up
  automatically.
- `clean-old-failed-jobs` - a `ClusterCleanupPolicy` that periodically removes failed jobs.
- `kyverno:generate-issuer` - a `ClusterRole` granting Kyverno permission to manage cert-manager `Issuer`
  resources, rendered only when both `rbac.authorization.k8s.io/v1` and `cert-manager.io/v1` are available.

## Root Value Files

| File                              | Purpose                                                             |
| ---------------------------------- | -------------------------------------------------------------------- |
| `values.yaml`                      | Default policy behavior, merged for every environment.              |
| `values-local.yaml`                | Relaxed resource limits and audit-only enforcement for local dev.    |
| `values-production.yaml`           | Audit-only policy enforcement for production clusters.               |
| `values-sf-k8s04-dev.yaml`         | Cluster-specific resource tuning for `sf-k8s04-dev`.                 |
| `values-subchart-overrides.yaml`   | Subchart image, RBAC, resource, and feature overrides.               |

## Chart Dependencies

This chart pulls in `kyverno` as a dependency. The version used is specified in `Chart.yaml` under `dependencies`.
If you change the version there, update the downloaded archive and commit the result together with the changed
`Chart.yaml` and `Chart.lock`:

```sh
 docker run \
   --rm \
   -u $(id -u) \
   -e HOME=/tmp \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm dependency update .
```

See the [Helm docs](https://helm.sh/docs/topics/charts/#chart-dependencies) for details. Renovate keeps this
dependency up to date automatically.

## Testing

### `values-subchart-overrides.yaml`

The `values-subchart-overrides.yaml` file is used to override values of the `kyverno` subchart. Values for the
subchart are kept separate from the values for the root chart so that unit tests can catch incompatible changes in
subchart values. This is necessary because Helm does not allow disabling the use of `values.yaml`; separating the
files makes it possible to test, for example, that the root chart and the subchart reference the same image
registry and repository.

### Run Helm Unittests

```sh
 docker run \
   --rm \
   -u $(id -u) \
   -e HELM_CACHE_HOME=/tmp/helm/.config \
   -v "$(pwd):/apps" \
   -w /apps \
   helmunittest/helm-unittest .
```

Or with output in JUnit format:

```sh
 docker run \
   --rm \
   -u $(id -u) \
   -e HELM_CACHE_HOME=/tmp/helm/.config \
   -v "$(pwd):/apps" \
   -w /apps \
   helmunittest/helm-unittest -o test-output.xml .
```

## Rendering Templates Locally

```sh
 docker run \
   --rm \
   -u $(id -u) \
   -e HOME=/tmp \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm template \
   --include-crds \
   --output-dir _local/local \
   --release-name kyverno \
   --skip-tests \
   -a batch/v1/CronJob \
   -a cert-manager.io/v1 \
   -a kyverno.io/v1 \
   -f values-subchart-overrides.yaml \
   -f values-local.yaml \
   -n kyverno \
   .
```

> [!NOTE]
> `-f` flags are applied in order, with later files overriding earlier ones. Keep
> `values-subchart-overrides.yaml` first and the environment-specific file last.

## Running The Pipeline Locally

Commands below assume the `SteadOps-Steadies-K8s-Workplace` workbench, which provides `helm`, `yq`, `kubectl`,
`hetzner-k3s`, and `act` directly on the shell.

To run the pipeline locally, start the workbench, `cd` into the folder containing this `README.md`, and run:

```sh
 act
```

On first execution you are asked which flavor of the `act` image should be used. The default `medium` flavor is a
good starting point.
