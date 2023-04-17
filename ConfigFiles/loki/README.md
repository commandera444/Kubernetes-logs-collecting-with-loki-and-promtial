# Required changes for Loki

Edit Loki configuration for storing Chunks and Indexes in AWS S3.

Add and change following configurations in default `values.yaml` file.

```yaml
limits_config:
    enforce_metric_name: false
    reject_old_samples: true
    reject_old_samples_max_age: 168h
    max_cache_freshness_per_query: 2m
    split_queries_by_interval: 2m
    max_entries_limit_per_query: 5000000


compactor:
    working_directory: /var/loki/compactor
    shared_store: s3
    compaction_interval: 10m 
    retention_enabled: true
limits_config:
    retention_period: 2160h
    max_query_length: 2200h
    max_entries_limit_per_query: 5000000


schema_config:
    configs:
    - from: "2023-03-29"
        store: boltdb-shipper
        object_store: s3
        schema: v11
        index:
        period: 24h
        prefix: loki_index_


storage_config:
    filesystem: null
    aws:
    region: <YOUR_REGION>
    bucketnames: <YOUR_BUCKET_NAME>
    s3forcepathstyle: false
    
    boltdb_shipper:
        active_index_directory: /var/loki/boltdb-shipper-active
        shared_store: s3
        cache_location: /var/loki/boltdb-shipper-cache
        cache_ttl: 24h


serviceAccount:
  create: true
  name: "loki"
  annotations:
    eks.amazonaws.com/role-arn: "<YOUR BUCKET ARN>"
  automountServiceAccountToken: true

compactor:
  # -- Specifies whether compactor should be enabled
  enabled: true

indexGateway:
  # -- Specifies whether the index-gateway should be enabled
  enabled: true
  
memcachedExporter:
  # -- Specifies whether the Memcached Exporter should be enabled
  enabled: true

memcachedChunks:
  # -- Specifies whether the Memcached chunks cache should be enabled
  enabled: true

memcachedFrontend:
  # -- Specifies whether the Memcached frontend cache should be enabled
  enabled: true

memcachedIndexQueries:
  # -- Specifies whether the Memcached index queries cache should be enabled
  enabled: true

memcachedIndexWrites:
  # -- Specifies whether the Memcached index writes cache should be enabled
  enabled: true
```             