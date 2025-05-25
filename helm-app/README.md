# helm

## Creating helm chart

```sh
helm create helm-app
```

Helm creates a chart scaffold with several folders and files:
- `Chart.yaml`
  - Contains metadata about the chart.
  - Usually you just need to increment the version whenever you update the chart.
- `charts/`
  - This is a (optional) folder where you store helm dependencies.
- `templates/`
  - Contains kubernetes manifest files that will be templated. (`deployment.yaml`, `service.yaml`, etc)
  - `NOTES.txt`: contains text displayed to user after chart has been installed, supports templating as well.
- `values.yaml`
  - Where we store our custom values.
- `.helmignore`
  - Files to ignore when templating

## Templating
- It makes charts configurable by injected values from `Values.yaml` into the manifest files in `templates/`
- It uses Go templating engine syntax
  - `{{ .Values.<path>.<to>.<value> }}`
  - Example: `{{ .Values.appName }}`
- The same syntax can be applied to `NOTES.txt`

## Deploying our app using Helm

Initial installation:
```sh
helm install mywebapp-release helm-app --values helm-app/values.yaml
```

Upgrading a release
```sh
helm upgrade mywebapp-release helm-app --values helm-app/values.yaml
```

## Templating for Multiple Env
- You can create env specific files overriding the default `Values.yaml`
- For example, you can create a `values-dev.yaml` that contains the values that you want to override (only), such as namespace, configmap data etc.

Then, deploy it using
```sh
helm install mywebapp-release helm-app --values helm-app/values.yaml -f helm-app/values-dev.yaml -n dev
```

## Teardown
```sh
# default namespace
helm uninstall mywebapp-release helm-app
# dev, and prod if you installed it
helm uninstall mywebapp-release helm-app -n dev
```
