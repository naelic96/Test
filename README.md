auth_enabled: false

server:
  http_listen_port: 3100

distributor:
  ring:
    kvstore:
      store: inmemory
  max_recv_msg_size_bytes: 10MB
  max_outstanding_per_tenant: 2000

ingester:
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
    final_sleep: 0s
  chunk_idle_period: 10m
  max_chunk_age: 1h
  chunk_target_size: 1572864 # ~1.5MB
  max_transfer_retries: 0
  max_chunk_series_limit: 100000
  max_chunks_per_query: 200000

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/index
    cache_location: /loki/cache
    shared_store: filesystem
  filesystem:
    directory: /loki/chunks

limits_config:
  enforce_metric_name: false
  max_streams_per_user: 50000
  max_query_lookback: 720h  # 30 giorni
  max_query_length: 24h
  max_entries_limit: 50000
  max_query_parallelism: 32
  max_chunks_per_query: 200000
  max_streams_per_query: 20000
  max_labels_per_series: 150
  max_label_name_length: 256
  max_label_value_length: 1024

query_range:
  split_queries_by_interval: 1h
  align_queries_with_step: true
  max_retries: 5
  cache_results: true
  results_cache:
    enable_fifocache: true
    fifocache:
      max_size_items: 10000
      validity: 10m

# Promozione label e rimozione service_name
labels_config:
  reject_labels:
    - service_name

# Ricezione OTLP gRPC
otelcol:
  protocols:
    grpc:
      listen_address: 0.0.0.0:4317
      max_recv_msg_size_mib: 10

# Ingest via Loki HTTP push API se serve
ingester_client:
  grpc_client_config:
    max_send_msg_size: 10485760

ruler:
  enable_api: true
  storage:
    type: local
    local:
      directory: /loki/rules
  alertmanager_url: http://alertmanager:9093

# Logger
server:
  log_level: info
