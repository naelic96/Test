auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9095
  log_level: info
  grpc_server_max_recv_msg_size: 41943040

distributor:
  ring:
    kvstore:
      store: inmemory

ingester:
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
  chunk_idle_period: 15m
  chunk_block_size: 262144
  max_chunk_age: 1h
  wal:
    enabled: true

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/index
    cache_location: /loki/cache
  filesystem:
    directory: /loki/chunks

schema_config:
  configs:
  - from: 2020-10-24
    store: boltdb-shipper
    object_store: filesystem
    schema: v13
    index:
      prefix: index_
      period: 24h

limits_config:
  ingestion_rate_mb: 20
  ingestion_burst_size_mb: 40
  max_streams_per_user: 50000
  max_global_streams_per_user: 200000
  max_entries_limit_per_query: 50000
  max_query_parallelism: 20
  max_query_lookback: 720h
  reject_old_samples: true
  reject_old_samples_max_age: 720h
  max_label_name_length: 1024
  max_label_value_length: 1024
  max_line_size: 256000

query_range:
  max_retries: 5
  align_queries_with_step: true

compactor:
  retention_enabled: true
  retention_delete_delay: 2h
  retention_delete_worker_count: 50

ruler:
  alertmanager_url: http://alertmanager:9093
  enable_api: true
  enable_alertmanager_v2: true
  evaluation_interval: 30s
