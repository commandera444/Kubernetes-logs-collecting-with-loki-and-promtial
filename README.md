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
            - cri: {}
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
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install loki grafana/loki-stack --values values.yaml
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
# `pattern` is in Grafana. In this example I'm sending `Multiline Logs` from Python App, where are `error` s in pattern parser and adding name for it. `<log>`

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


# You can also make your log alerting High Avilibility.
# For this you can yous AWS S3 Bucket or GCS DynamoDB or other methods.
# You can see more about it in Documentation page.
# https://grafana.com/docs/loki/latest/operations/storage/
# For High Avilibilty you must change some COnfigurations in YAML file.
# In this example I'm using AWS S3 Bucket
# And for Grafana Dashboards and Alerts I'm using custom Dashboard JSON code and Alerts and Notifications JSON Code.
```yaml
loki:
  enabled: true
  config:
    common:
      storage: 
        filesystem: null
        s3:
          s3: s3://<aws-region-name>/<your-bucket-name>
          endpoint: <aws-endpoint>
          insecure: false
          bucketnames: <your-bucket-name>
          access_key_id: <your-access-key-id>
          secret_access_key: <your-secret-access-key-id>
          s3forcepathstyle: true

    compactor:
      working_directory: /data/my/compactor
      shared_store: s3
      compaction_interval: 10m 
      retention_enabled: true
    limits_config:
      retention_period: 744h
      max_query_length: 800h
    
    ingester: 
      chunk_block_size: 262144
      chunk_idle_period: 3m
      chunk_retain_period: 1m 
      lifecycler:
        ring: 
          replication_factor: 1
      max_transfer_retries: 0
      wal: 
        dir: /data/loki/wal

    schema_config:
      configs:
        - from: "2023-03-21"
          store: boltdb-shipper
          object_store: s3
          schema: v11
          index:
            period: 24h
            prefix: loki_index_
    storage_config:
      filesystem: null
      aws:
        s3: s3://<aws-region-name>/<your-bucket-name>
        s3forcepathstyle: true
        insecure: false
        bucketnames: <your-bucket-name>
        access_key_id: <your-access-key-id>
        secret_access_key: <your-secret-access-key-id>
        
      boltdb_shipper:
          active_index_directory: /data/loki/boltdb-shipper-active
          shared_store: s3
          cache_location: /data/loki/boltdb-shipper-cache
          cache_ttl: 24h

###This Configuration will be collect your Indexes and log parts from your Cluster daily, collect those and send to AWS S3 Bucket every 3 minutes.
###In this Configuration you can see your Cluster logs from grafana Dashboard. You can see logs within 30days.
###Queries for this period will taken from AWS S3 Bucket.
###Don't worry about alerts. Alerts are working in real time period.
###Ligs well be saved in S3 as indexes and Bin. files.
###Compactor will be commpress All logs in once everey 10 minutes.
###You can also see more about configurations in Documentation page
###https://grafana.com/docs/loki/latest/operations/storage/retention/
###https://grafana.com/docs/loki/latest/operations/storage/schema/
###https://grafana.com/docs/loki/latest/operations/storage/boltdb-shipper/



promtail:
  enabled: true
  config:
    snippets:
      scrapeConfigs: |
        - job_name: custom-config
          pipeline_stages:
            - docker: {}
            - cri: {}
            - multiline:
                firstline: '^\['
                max_wait_time: 3s
            - regex:
                expression: '(?m)^.*$'
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
# I converted my Dashboard, Notification Policy, and Notification Chanel from previous step in to JSON/YAML format.
# After then you must create COnfigMaps And Deployment For Grafana.
# I'm using Grafana File Provisioning.
# For converting you can use GRAFANA API KEYS, and make requests for your Dashboard and Alert Configs JSON/YAML model, or Export it from GRAFANA Dashboard.
# You can see more about it in Documentation page.
# https://grafana.com/docs/grafana/latest/administration/api-keys/
# https://grafana.com/docs/grafana/latest/developers/http_api/
# https://grafana.com/docs/grafana/latest/alerting/set-up/provision-alerting-resources/file-provisioning/
# For Example
```yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard
data:
  dashboard.json: |
    {
      "annotations": {
        "list": [
          {
            "builtIn": 1,
            "datasource": {
              "type": "grafana",
              "uid": "-- Grafana --"
            },
            "enable": true,
            "hide": true,
            "iconColor": "rgba(0, 211, 255, 1)",
            "name": "Annotations & Alerts",
            "target": {
              "limit": 100,
              "matchAny": false,
              "tags": [],
              "type": "dashboard"
            },
            "type": "dashboard"
          }
        ]
      },
      "editable": true,
      "fiscalYearStartMonth": 0,
      "graphTooltip": 0,
      "id": 2,
      "links": [],
      "liveNow": false,
      "panels": [
        {
          "datasource": {
            "type": "loki",
            "uid": "P982945308D3682D1"
          },
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 0
          },
          "id": 4,
          "options": {
            "dedupStrategy": "none",
            "enableLogDetails": true,
            "prettifyLogMessage": false,
            "showCommonLabels": false,
            "showLabels": false,
            "showTime": false,
            "sortOrder": "Descending",
            "wrapLogMessage": false
          },
          "targets": [
            {
              "datasource": {
                "type": "loki",
                "uid": "P982945308D3682D1"
              },
              "editorMode": "builder",
              "expr": "{app=\"web\",namespace=\"webapps\"}",
              "queryType": "range",
              "refId": "A"
            }
          ],
          "title": "Logs Python",
          "transparent": true,
          "type": "logs"
        },
        {
          "datasource": {
            "type": "loki",
            "uid": "P982945308D3682D1"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 9,
            "w": 12,
            "x": 12,
            "y": 0
          },
          "id": 2,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "single",
              "sort": "none"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "loki",
                "uid": "P982945308D3682D1"
              },
              "editorMode": "code",
              "expr": "rate({app=\"web\",namespace=\"webapps\"} |= \"error\" | pattern `<logs>` [1m])",
              "queryType": "range",
              "refId": "A"
            }
          ],
          "title": "Python Dash",
          "transparent": true,
          "type": "timeseries"
        }
      ],
      "refresh": "",
      "revision": 1,
      "schemaVersion": 38,
      "style": "dark",
      "tags": [],
      "templating": {
        "list": []
      },
      "time": {
        "from": "now-2d",
        "to": "now"
      },
      "timepicker": {},
      "timezone": "",
      "title": "PythonDash",
      "uid": "obwQYaf4k",
      "version": 4,
      "weekStart": ""
    }{
      "annotations": {
        "list": [
          {
            "builtIn": 1,
            "datasource": {
              "type": "grafana",
              "uid": "-- Grafana --"
            },
            "enable": true,
            "hide": true,
            "iconColor": "rgba(0, 211, 255, 1)",
            "name": "Annotations & Alerts",
            "target": {
              "limit": 100,
              "matchAny": false,
              "tags": [],
              "type": "dashboard"
            },
            "type": "dashboard"
          }
        ]
      },
      "editable": true,
      "fiscalYearStartMonth": 0,
      "graphTooltip": 0,
      "id": 2,
      "links": [],
      "liveNow": false,
      "panels": [
        {
          "datasource": {
            "type": "loki",
            "uid": "P982945308D3682D1"
          },
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 0
          },
          "id": 4,
          "options": {
            "dedupStrategy": "none",
            "enableLogDetails": true,
            "prettifyLogMessage": false,
            "showCommonLabels": false,
            "showLabels": false,
            "showTime": false,
            "sortOrder": "Descending",
            "wrapLogMessage": false
          },
          "targets": [
            {
              "datasource": {
                "type": "loki",
                "uid": "P982945308D3682D1"
              },
              "editorMode": "builder",
              "expr": "{app=\"web\",namespace=\"webapps\"}",
              "queryType": "range",
              "refId": "A"
            }
          ],
          "title": "Logs Python",
          "transparent": true,
          "type": "logs"
        },
        {
          "datasource": {
            "type": "loki",
            "uid": "P982945308D3682D1"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 9,
            "w": 12,
            "x": 12,
            "y": 0
          },
          "id": 2,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "single",
              "sort": "none"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "loki",
                "uid": "P982945308D3682D1"
              },
              "editorMode": "code",
              "expr": "rate({app=\"web\",namespace=\"webapps\"} |= \"error\" | pattern `<logs>` [1m])",
              "queryType": "range",
              "refId": "A"
            }
          ],
          "title": "Python Dash",
          "transparent": true,
          "type": "timeseries"
        }
      ],
      "refresh": "",
      "revision": 1,
      "schemaVersion": 38,
      "style": "dark",
      "tags": [],
      "templating": {
        "list": []
      },
      "time": {
        "from": "now-2d",
        "to": "now"
      },
      "timepicker": {},
      "timezone": "",
      "title": "PythonDash",
      "uid": "obwQYaf4k",
      "version": 4,
      "weekStart": ""
    }

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-provisioning
data:
  provisioning.yaml: |

      - name: Python
        orgId: 1
        folder: 'Python'
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /tmp/dashboards/Python

--- 

apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-provisioning-notifier
data:
  notifiers.json: |
      apiVersion: 1
      groups:
          - orgId: 1
            name: WebApps
            folder: Python
            interval: 30s
            rules:
              - uid: P982945308D3682D1
                title: Python Errors
                condition: B
                data:
                  - refId: A
                    queryType: range
                    relativeTimeRange:
                      from: 60
                      to: 0
                    datasourceUid: P982945308D3682D1
                    model:
                      editorMode: code
                      expr: rate({app="web",namespace="webapps"} |= "error" | pattern `<logs>` [1m] )
                      hide: false
                      intervalMs: 1000
                      maxDataPoints: 43200
                      queryType: range
                      refId: A
                  - refId: B
                    relativeTimeRange:
                      from: 60
                      to: 0
                    datasourceUid: __expr__
                    model:
                      conditions:
                          - evaluator:
                              params: []
                              type: gt
                            operator:
                              type: and
                            query:
                              params:
                                  - B
                            reducer:
                              params: []
                              type: last
                            type: query
                      datasource:
                          type: __expr__
                          uid: __expr__
                      expression: A
                      hide: false
                      intervalMs: 1000
                      maxDataPoints: 43200
                      reducer: last
                      refId: B
                      type: reduce
                noDataState: OK
                execErrState: Alerting
                for: 30s
                isPaused: false

      contactPoints:
        - orgId: 1
          name: slack
          receivers:
          - uid: FBkGbfBVz
            type: slack
            isDefault: true
            settings:
              recipient: "C04UQBTRXUG"
              token: "xoxb-4697024486131-4805827928644-Gl5QJk7sWl6GuE214NEAYIx8"
              title: |
                ERRORS.
              text: |
                {{ template "slack.message" . }}
      templates:
        - orgID: 1
          name: My
          template: |
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


      policies:
        - orgId: 1
          receiver: slack
          group_by:
          - alertname
          group_wait: 30s
          group_interval: 30s
          repeat_interval: 1m


--- 

apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-provisioning-datasources
data:
  datasources.yaml: |
      apiVersion: 1
      datasources:
      - name: loki
        uid: P982945308D3682D1
        type: loki
        isDefault: true
        access: proxy
        url: http://loki:3100






---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana-deploy
  labels:
    app: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 25%
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: grafana
    spec:
      containers:
        - image: grafana/grafana:9.4.3
          name: grafana
          volumeMounts:
          - name: grafana-dashboard-volume
            mountPath: /tmp/dashboards/Python
          - name: grafana-provisioning-volume
            mountPath: /etc/grafana/provisioning/dashboards
            readOnly: true
          - name: grafana-provisioning-notifier
            mountPath: /etc/grafana/provisioning/alerting
          - name: grafana-provisioning-datasources
            mountPath: /etc/grafana/provisioning/datasources
          ports:
          - containerPort: 3000
            name: grafana
            protocol: TCP
      volumes:
      - name: grafana-dashboard-volume
        configMap:
          name: grafana-dashboard
      - name: grafana-provisioning-volume
        configMap:
          name: grafana-provisioning
      - name: grafana-provisioning-notifier 
        configMap:
          name: grafana-provisioning-notifier
      - name: grafana-provisioning-datasources
        configMap:
          name: grafana-provisioning-datasources
```
