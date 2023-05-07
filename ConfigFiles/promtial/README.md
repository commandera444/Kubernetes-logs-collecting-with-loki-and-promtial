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
            - cri: {}
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
                  - <YOUR SPECIFIC NAMESPACE NAME>
                  - <YOUR SPECIFIC NAMESPACE NAME>               
          relabel_configs:
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
              replacement: /var/log/pods/*$1/*.log
              regex: true/(.*)
              separator: /
              source_labels:
                - __meta_kubernetes_pod_annotationpresent_kubernetes_io_config_hash
                - __meta_kubernetes_pod_annotation_kubernetes_io_config_hash
                - __meta_kubernetes_pod_container_name
              target_label: __path__   

```
