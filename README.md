auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096
  log_level: info
  log_format: json

ingester:
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
    final_sleep: 0s
  chunk_idle_period: 10m
  max_chunk_age: 1h
  chunk_target_size: 1048576
  chunk_retain_period: 30s
  max_transfer_retries: 0

distributor:
  ring:
    kvstore:
      store: inmemory
    replication_factor: 1

ingester_client:
  grpc_client_config:
    max_send_msg_size: 10485760
    max_recv_msg_size: 10485760

query_range:
  align_queries_with_step: true
  max_retries: 5
  split_queries_by_interval: 15m
  max_query_length: 24h
  parallelise_shardable_queries: true
  cache_results: true
  results_cache:
    enabled: true
    cache:
      filesystem:
        directory: /tmp/loki-cache
    ttl: 2h
    max_size_mb: 1000

frontend_worker:
  frontend_address: 127.0.0.1:9095
  grpc_client_config:
    max_send_msg_size: 10485760
    max_recv_msg_size: 10485760

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/index
    cache_location: /loki/cache
    shared_store: filesystem
  filesystem:
    directory: /loki/chunks

schema_config:
  configs:
    - from: 2023-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  enforce_metric_name: false
  max_streams_per_user: 500000
  max_streams_per_query: 10000
  max_entries_limit_per_query: 100000
  max_chunks_per_query: 2000
  max_chunk_age: 1h
  max_query_length: 24h
  ingestion_rate_mb: 50
  ingestion_burst_size_mb: 100
  max_label_name_length: 1024
  max_label_value_length: 4096
  max_labels_per_series: 50

chunk_store_config:
  max_look_back_period: 336h

table_manager:
  retention_deletes_enabled: true
  retention_period: 720h

ruler:
  enable_api: true
  alertmanager_url: http://alertmanager:9093

receivers:
  otlp:
    protocols:
      grpc:
        max_recv_msg_size_mib: 20
      http:
        max_recv_msg_size_mib: 20

prometheus:
  wal_directory: /loki/wal

label_configs:
  - source_labels: [__resource_aws_region]
    target_label: aws_region
  - source_labels: [__resource_global_app]
    target_label: global_app
