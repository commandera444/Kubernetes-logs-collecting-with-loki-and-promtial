# Required changes for Promtail

Edit Promtail configuration for collecting logs from Cluster with specific namespaceces and enabling Multiline logging.

Add and change following configurations in default `values.yaml` file.


```yaml
#Line 347
clients:
- url: http://loki-loki-distributed-gateway.<YOUR NAMESPACE>.svc.cluster.local/loki/api/v1/push

#Line 418

    scrapeConfigs: |
      - job_name: custom-config
        pipeline_stages:
          - docker: {}
          - regex:
              expression: "^(?s)(?P<time>\\S+?) (stdout|stderr) (\\S+?) (?P<content>.*)$"
          - timestamp:
              source: time
              format: RFC3339Nano
          - output:
              source: content
          - multiline:
              firstline: '(^INFO)|(Beginning Flow run)|(\{\"t\")'
              max_wait_time: 3s
              max_lines: 1024
        kubernetes_sd_configs:
          - role: pod
            namespaces:
              names: 
            #    - <YOUR SPECIFIC NAMESPACES>
            #    - <YOUR SPECIFIC NAMESPACES>
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
              - __meta_kubernetes_pod_controller_kind
            regex: ^;*([^;]+)(;.*)?$
            action: replace
            target_label: controller
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