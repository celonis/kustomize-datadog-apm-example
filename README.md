# Demo: Datadog APM configuration using kustomize
This repository contains sample code for using [kustomize](https://kustomize.io/) to configure [Datadog APM](https://www.datadoghq.com/product/apm/). This was presented as *Quick Bite* at [Dash 2020](https://www.dashcon.io/).

First, define a place to work:
```
DEMO_HOME=$(mktemp -d)
```
Create a kustomize base containing a simple deployment:
```
BASE=$DEMO_HOME
mkdir -p $BASE
cat <<EOF >$BASE/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
EOF

cat <<EOF >$BASE/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: foo
  labels:
    app: foo
spec:
  replicas: 1
  template:
    metadata:
      name: foo
      labels:
        app: foo
    spec:
      containers:
      - name: foo
        image: my-java-app
        env: []
        envFrom: []
  selector:
    matchLabels:
      app: foo
EOF
```
If you now run `kustomize build $DEMO_HOME` you will see the deployment manifest rendered without any changes.

Now, "import" the remote transformers for applying the configuration changes:
```
cat <<EOF >>$BASE/kustomization.yaml
transformers:
- github.com:celonis/kustomize-datadog-apm-example
```
If you run `kustomize build $DEMO_HOME` again, you will see the following changes have been applied to the deployment manifest:

- An init container is added to your deployment, which uses `curl` to download the Datadog Java APM agent into an `emptyDir` volume shared between the init and application container
- All environment variables from a config map named `datadog-apm-env` are mounted into the app container (this is not really required, but useful for adding some environment specific configuration)
- The `JDK_JAVA_OPTIONS` environment variable is set in order to attach the APM agent at runtime
- The `DD_AGENT_HOST` environment variable points to the host IP using the downward API
- The `DD_SERVICE_NAME` contains the value of deployment's `app` label 

## Advanced Usage
Here are some advanced patterns which are useful when productionizing this approach for your team.
### Versioning
When importing the remote transformers, you can also target a specific revisions. For example, check out branch `release-1.1` which adds some configuration for logs injection and service name mapping:

```yaml
transformers:
- github.com:celonis/kustomize-datadog-apm-example?ref=release-1.1
```

Or, if you want to make sure you don't run into any breaking changes in the configuration, stick to the initial v1.0.0 release:
```yaml
transformers:
- github.com:celonis/kustomize-datadog-apm-example?ref=v1.0.0
```

## Questions?
If anything is unclear and you need some help, feel free to [create an issue](https://github.com/celonis/kustomize-datadog-apm-example/issues/new).
