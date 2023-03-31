# This is a `values.yaml` file little description.

# About LOKI configuration.


In this section you configure your AWS S3 Bucket as Storage for saving Indexes and Chunkes. More about in [Grafana Documentation Page.](https://grafana.com/docs/loki/latest/operations/storage/filesystem/)
```yaml
    storage_config:
      filesystem: null
      aws:
        region: <AWS_REGION>
        s3forcepathstyle: true
        insecure: false
        bucketnames: <YOUR_BUCKET_NAME>
        access_key_id: <YOUR_ACCESS_KEY_ID>
        secret_access_key: <YOUR_SECRET_KEY>

```


In this section you configure detalis for Indexes and Chunkes period, and At what time will loki start save and sending CHunks and Indexes. More about in [Grafana Documentation Page.](https://grafana.com/docs/loki/latest/operations/storage/schema/)
```yaml
    schema_config:
      configs:
        - from: "2023-03-21"
          store: boltdb-shipper
          object_store: s3
          schema: v11
          index:
            period: 24h
            prefix: loki_index_
```            


The ingester service is responsible for writing log data to long-term storage backends. 
For example if Cunks size in period will be 2MB, ingester will be send it in S3 Bucket, if after 30 minutes Chunks size will not be a 2MB ingester also will be send Chunks in S3 Bucket. More about in [Grafana Documentation Page.](https://grafana.com/docs/loki/latest/operations/storage/boltdb-shipper/#ingesters)
```yaml
    ingester: 
      chunk_encoding: gzip
      chunk_target_size: 2000000
      chunk_idle_period: 30m
      chunk_retain_period: 0s 
      lifecycler:
        ring: 
          replication_factor: 1
      max_transfer_retries: 0
      wal: 
        dir: /data/loki/wal
```        


The compactor will be compact all Indexes in one file every 10 minutes and will be send it in S3 Bucket.
Retention period and Max query lenght allowed you to view Indexes and Chunkes from S3 Bucket from some period.
In this example Max 30 days.  More about in [Grafana Documentation Page.](https://grafana.com/docs/loki/latest/operations/storage/retention/#retention-configuration)
```yaml
    compactor:
      working_directory: /data/my/compactor
      shared_store: s3
      compaction_interval: 10m 
      retention_enabled: true
    limits_config:
      retention_period: 744h
      max_query_length: 800h
```


# About PROMTAIL configuration.
In this file I'm using default configuration.
I'm only add `PipeLine Stages` for `MultiLine` logs.


For Multiline logs searching I'm using "firsline:" expression, and for compacting all lines I'm using "-regex:" More about in [Grafana Documentation Page.](https://grafana.com/docs/loki/latest/clients/promtail/stages/multiline/)
```yaml 
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
```                