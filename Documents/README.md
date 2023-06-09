# This is a `values.yaml` file little description.

# About LOKI configuration.


In this section you configure your AWS S3 Bucket as Storage for saving Indexes and Chunkes. More about in [Grafana Loki Documentation Page.](https://grafana.com/docs/loki/latest/operations/storage/filesystem/)
```yaml
    storage_config:
        filesystem: null
        aws:
        region: <YOUR_REGION>
        bucketnames: <YOUR_BUCKET_NAME>
        s3forcepathstyle: false

```


In this section you configure detalis for Indexes and Chunkes period, and At what time will loki start save and sending Chunks and Indexes. More about in [Grafana Loki Documentation Page.](https://grafana.com/docs/loki/latest/operations/storage/schema/)
```yaml
  schema_config:
      configs:
      - from: "2023-03-29"
          store: boltdb-shipper
          object_store: s3
          schema: v11
          index:
          period: 24h
          prefix: loki_index_
```            


The ingester service is responsible for writing log data to long-term storage backends. 
For example if Cunks size in period will be 262144KB, ingester will be send it in S3 Bucket, if after 30 minutes Chunks size will not be a 262144KB ingester also will be send Chunks in S3 Bucket. More about in [Grafana Loki Documentation Page.](https://grafana.com/docs/loki/latest/operations/storage/boltdb-shipper/#ingesters)
```yaml
    ingester:
      lifecycler:
        ring:
          kvstore:
            store: memberlist
          replication_factor: 1
      chunk_idle_period: 30m
      chunk_block_size: 262144
      chunk_encoding: snappy
      chunk_retain_period: 1m
      max_transfer_retries: 0
      wal:
        dir: /var/loki/wal
```        


The compactor will be compact all Indexes in one file every 10 minutes and will be send it in S3 Bucket.
Retention period and Max query lenght allowed you to view Indexes and Chunkes from S3 Bucket from some period.
In this example Max 90 days.  More about in [Grafana Loki Documentation Page.](https://grafana.com/docs/loki/latest/operations/storage/retention/#retention-configuration)
```yaml
    compactor:
      working_directory: /data/my/compactor
      shared_store: s3
      compaction_interval: 10m 
      retention_enabled: true
limits_config:
    retention_period: 2160h
    max_query_length: 2200h
    max_entries_limit_per_query: 5000000
```


# About PROMTAIL configuration.
There is special configuration for collectiong logs and enabling Multiline stages.


More about in [Grafana Promtail Documentation Page.](https://grafana.com/docs/loki/latest/clients/promtail/stages/multiline/)
```yaml 
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