# Kyverno helm deployment

Repository containing manifests for [kyverno](https://github.com/kyverno/kyverno/).
Never install the content of this repo on our clusters manually. This is all done by argocd.

## Dependencies

This chart pulls in `kyverno` as a dependency. The version
used is specified in `Chart.yaml` in the `dependencies` section.
If you change the version in there, you need to then run

    $ helm dependency update

in order to have the chart downloaded to the `charts` directory
and then also commit that new version alongside with the altered
`Chart.yaml` file.

See the [Helm docs](https://helm.sh/docs/topics/charts/#chart-dependencies)
for details.

## Render resource local

```
  helm template -n argocd --release-name applications-root --include-crds --skip-tests \
  -a cert-manager.io/v1 \
  -a batch/v1/CronJob \
  -a kyverno.io/v1 \
  -f values-local.yaml --output-dir _local .
```
