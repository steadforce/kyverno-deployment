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

# Testing

## values-subchart-overrides.yaml

The `values-subchart-overrides.yaml` file is used to override values in the postgres-operator chart.
We have to separate the values for the subcharts from the values for the main chart, to be able to
unit test for incompatible changes in values of the subcharts. This is necessary because helm does not allow
switching off the usage of values.yaml. Now it's possible to test if we use the same registry and repository
for images as the subcharts are using.

## run helm unittests

```shell
 docker run --pull=always -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest .
```

Or with output in JUnit format:

```shell
 docker run --pull=always -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest -o test-output.xml .
```

## Render resource local

```
  helm template \
  --include-crds \
  --output-dir _local/local \
  --release-name kyverno \
  --skip-tests \
  -a cert-manager.io/v1 \
  -a batch/v1/CronJob \
  -a kyverno.io/v1 \
  -f values-subchart-overrides.yaml \
  -f values-local.yaml \
  -n kyverno \
  .
```

## Run act pipeline locally

To run the pipeline in local environment, start up the workbench, cd into the folder containing this
`README.md` and execute the following command:

```shell
  act
```

On first execution you're asked which flavour of the act image should be used. Using the default `medium`
is a good starting point.
