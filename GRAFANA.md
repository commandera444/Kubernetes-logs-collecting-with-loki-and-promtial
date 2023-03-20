This is a documentation how to install Grafana Loki Promtail and how to Config Simple Alerts For Python Application in Kubernetes Cluster.



# 1. Create a `values.yaml` file.
# 2. Add there specail configuration for `Promtail`, `Loki` and`Grafana` for `Multiline` logging.
#    `values.yaml`

``` yaml
grafana:
  enabled: true
  sidecar:
    datasources:
      enabled: true
    dashboards:
      enabled: true
      label: grafana_dashboard
      folder: /test/dashboards
      provider:
        foldersFromFilesStructure: true
  image:
    tag: 9.4.3

loki:
  enabled: true

#Special Configuration Promtile For Enabling Multiline in Pip_Line Stages.
#You Can Change Your Container Run Time To Other, I'm Using Docker.
promtail:
  enabled: true
  config:
    snippets:
      scrapeConfigs: |
        - job_name: custom-config
          pipeline_stages:
            - docker: {}
            #Special Lines For Logs (First Line Searching and Commbinong All Characters and Letters)
            - multiline:
                firstline: '^\['
                max_wait_time: 3s
            - regex:
                expression: '(?m)^.*$'
            #Special Line End.
            - json:
                expressions:
                  timestamp: timestamp
                  level: level
                  thread: thread
                  class: logger
                  message: message
                  context: context
            - labels:
                level:
                class:
                context:
                thread:
            - timestamp:
                format: RFC3339Nano
                source: timestamp
            - output:
                source: message
          kubernetes_sd_configs:
            - role: pod
          relabel_configs:
            - source_labels:
                - __meta_kubernetes_pod_controller_name
              regex: ([0-9a-z-.]+?)(-[0-9a-f]{8,10})?
              action: replace
              target_label: __tmp_controller_name
            - source_labels:
                - __meta_kubernetes_pod_label_app_kubernetes_io_name
                - __meta_kubernetes_pod_label_app
                - __tmp_controller_name
                - __meta_kubernetes_pod_name
              regex: ^;*([^;]+)(;.*)?$
              action: replace
              target_label: app
            - source_labels:
                - __meta_kubernetes_pod_label_app_kubernetes_io_instance
                - __meta_kubernetes_pod_label_release
              regex: ^;*([^;]+)(;.*)?$
              action: replace
              target_label: instance
            - source_labels:
                - __meta_kubernetes_pod_label_app_kubernetes_io_component
                - __meta_kubernetes_pod_label_component
              regex: ^;*([^;]+)(;.*)?$
              action: replace
              target_label: component
            - action: replace
              source_labels:
              - __meta_kubernetes_pod_node_name
              target_label: node_name
            - action: replace
              source_labels:
              - __meta_kubernetes_namespace
              target_label: namespace
            - action: replace
              replacement: $1
              separator: /
              source_labels:
              - namespace
              - app
              target_label: job
            - action: replace
              source_labels:
              - __meta_kubernetes_pod_name
              target_label: pod
            - action: replace
              source_labels:
              - __meta_kubernetes_pod_container_name
              target_label: container
            - action: replace
              replacement: /var/log/pods/*$1/*.log
              separator: /
              source_labels:
              - __meta_kubernetes_pod_uid
              - __meta_kubernetes_pod_container_name
              target_label: __path__
            - action: replace
              regex: true/(.*)
              replacement: /var/log/pods/*$1/*.log
              separator: /
              source_labels:
              - __meta_kubernetes_pod_annotationpresent_kubernetes_io_config_hash
              - __meta_kubernetes_pod_annotation_kubernetes_io_config_hash
              - __meta_kubernetes_pod_container_name
              target_label: __path__
```
# 3. Next apply in Kubernetes using Helm

``` bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install loki-stack grafana/loki-stack --values values.yaml
```
# 4. Now We can connect to Grafana interface Using Your browser.
# We can connect to Grafana Using Ingress or Port Forward or other methods in Kubernetes.
# Deafult Login For Grafana is `admin`
# For getting password You must do this command in Your Terminal
```bash
k get secrets <-n YOUR_NAME_SPACE_NAME_IF_YOU_INSTALLED_APP_THERE> loki-stack-grafana -o jsonpath="{.data.admin-password}" |base64 -d
```
![Tux, the Linux mascot](/Screens/Sources.png)