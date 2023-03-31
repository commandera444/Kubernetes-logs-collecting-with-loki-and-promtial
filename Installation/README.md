# Installation.
First add helm repository.

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install loki grafana/loki-stack --values values.yaml -n <YOUR_NAMESPACE_NAME> --create-namespace
```
I already made `values.yaml` and other configurations.
See more about description in [Documentation](.././Documents/).
To get default password for web page you can use this command. The default username is `admin`.
```bash
k get secrets -n <YOUR_NAMESPACE_NAME> loki-stack-grafana -o jsonpath="{.data.admin-password}" |base64 -d
```
# Next step.
Now you can view your Kubernetes applications logs.
See more [in this page](.././LogsView/)