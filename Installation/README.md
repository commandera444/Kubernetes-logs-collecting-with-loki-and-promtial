# Installation.
* Before starting installation make your `values.yaml` files. 
* See about required changes [here.](../ConfigFiles/README.md)


First add helm repository.

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install loki  grafana/loki-distributed --values values.yaml  -n <YOUR_NAMESPACE_NAME> --create-namespace
helm install  promtail -n <YOUR_NAMESPACE_NAME> grafana/promtail --values values.yaml
helm install grafana -n <YOUR_NAMESPACE_NAME> grafana/grafana 

```

# Next step.
Now you can view your Kubernetes applications logs.
See more [in this page.](.././LogsView/README.md)