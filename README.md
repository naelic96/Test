auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9095
  log_level: info
  grpc_server_max_recv_msg_size: 41943040    # 40MB per gRPC msg
  grpc_server_max_send_msg_size: 41943040

distributor:
  ring:
    kvstore:
      store: inmemory
  max_outstanding_per_tenant: 2000

ingester:
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
  chunk_idle_period: 15m
  chunk_block_size: 262144       # 256KB chunk blocks
  max_chunk_age: 1h
  max_chunks_per_query: 200000
  max_chunk_series_limit: 1000000
  max_transfer_retries: 5
  wal:
    enabled: true

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/index
    cache_location: /loki/cache
    shared_store: filesystem
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
  max_query_lookback: 720h  # 30 giorni
  reject_old_samples: true
  reject_old_samples_max_age: 720h
  max_label_name_length: 1024
  max_label_value_length: 1024
  max_line_size: 256000
  max_entries_per_push: 5000

query_range:
  split_queries_by_interval: 15m
  max_retries: 5
  align_queries_with_step: true

query_frontend:
  max_outstanding_per_tenant: 1000
  compress_responses: true

ruler:
  alertmanager_url: http://alertmanager:9093
  enable_api: true
  enable_alertmanager_v2: true
  evaluation_interval: 30s

# Promozione delle label (limita le label per ottimizzare)
labels_config:
  label_limits:
    max_labels_per_stream: 20
  drop_labels:
    - service_name

# OTLP receiver integrato Loki (serve per ricevere logs OTLP/gRPC)
receivers:
  otlp:
    protocols:
      grpc:
        max_recv_msg_size_mib: 40
      http:

# Configurazione pipeline di ricezione logs (separata in Loki 3.5)
ingestion:
  receivers:
    otlp:
      protocols:
        grpc:
          max_recv_msg_size_mib: 40

# Caches (memcache disabilitato)
# (non configurato, usa defaults)

# Logs retention 30 giorni (puoi adattare a tuoi volumi e spazio)
compactor:
  retention_enabled: true
  retention_delete_delay: 2h
  retention_delete_worker_count: 50
  retention_period: 720h

# Storage backend retention impostata in compactor + schema

# Log level di Loki per debug:
logging:
  level: info
