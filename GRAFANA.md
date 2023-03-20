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
# 5. Go To Datasources field and check Is your Loki Data Source working correctly.
![Sources](/Screens/Sources.png)![Sources](/Screens/Loki.png)

# Its OK.
![Sources](/Screens/OK.png)

# 6. Next go to the Dashboard and create Folder for Application Loging.
![Sources](/Screens/Folder.png)
# 7. After Then Craete a new Dashboard in this Folder.
# Select Datasource LOKI and add Your Query For This Panell
# For Example I'm Using Query For Python Application (Searching "error" in logs every 1 minutes).
```json
rate({app="web",namespace="webapps"} |= "error" | pattern `<log>` [1m] )
```
![Sources](/Screens/Query.png)
# `pattern` is `PARSER` in Grafana. You can read more about parsers in documentation page. In this example I'm sending `Multiline Logs` from Python App, where are `error` s in pattern parser and adding name for it. `<log>`

# 8. After You must add `Notification Policy`, `Contact Point` and `Notification templates` for Alerting. 
# I'm using Slack.
# You must eddit `optional Slack settings`  for sneding Correct Message.
![Sources](/Screens/CP.png)![Sources](/Screens/Slack.png)![Sources](/Screens/SlackSpecial.png)
# Add `Title` 
```json
ERRORS.
```
# Add Text Body
```json
{{ template "slack.message" . }}
```
# 9. Next add `Notification templates`.
![Sources](/Screens/TMP.png)
# Add SPECIAL CONFIG FOR NORMAL ALERT MESSAGES.
```json
{{ define "slack.print_alert" -}}
[{{.Status}}]
Labels:
{{ range .Labels.SortedPairs -}}
- {{ .Name }}: {{ .Value }}
{{ end -}}
{{- end }}
{{ define "slack.message" -}}
{{ if .Alerts.Firing -}}
{{ len .Alerts.Firing }} firing alert(s):
{{ range .Alerts.Firing }}
{{ template "slack.print_alert" . }}
{{ end -}}
{{ end }}
{{- end }}
```
# 10. Now go back to Your Dashboard and add Alert Configuration for Your Query.
![Sources](/Screens/Alert1.png)![Sources](/Screens/Alert2.png)![Sources](/Screens/Alert3.png)
# 11. Now, when your Application returnet error messages in logs, your Alert manager sent Alert to Your Slack Chanel.
# Its look like this.
![Sources](/Screens/AlertMessage.png)
# 12. You also can see Your Application Logs in Dashboard .
# Add New Panel In Dashboard, Add Your `Query`, Select `Panel type` `Logs` and see your Application Logs in Grafana Interface. 
# If log type is `Miltiline`, then you'll be see logs with `Multiline` type.
# If log type is `Single Line`, then you'll be see logs with `Single Line` type.
# `MultiLine`
![Sources](/Screens/MultiLine.png)
# `SingleLine`
![Sources](/Screens/SingleLine.png)