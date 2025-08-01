auth_enabled: false

server:
  http_listen_port: 3100
  log_level: info

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
    final_sleep: 0s
  chunk_idle_period: 10m
  max_chunk_age: 1h
  chunk_target_size: 1048576  # 1MB

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/index
    cache_location: /loki/cache
    shared_store: filesystem
  filesystem:
    directory: /loki/chunks

limits_config:
  max_streams_per_user: 50000
  max_query_lookback: 720h   # 30 giorni
  max_query_length: 24h
  max_label_name_length: 256
  max_label_value_length: 1024

query_range:
  align_queries_with_step: true
  max_retries: 5
  split_queries_by_interval: 1h

ruler:
  enable_api: true
  storage:
    type: local
    local:
      directory: /loki/rules
  alertmanager_url: http://alertmanager:9093

ingester_client:
  grpc_client_config:
    max_send_msg_size: 10485760  # 10MB

# Configurazione per promuovere o scartare label si fa con:
# relabel_configs (se usi pipeline o config di scrape lato collector/alloy)
# Loki 3.5 NON supporta labels_config

# Per OTLP ricezione, config esterna OTEL Collector/Alloy invia a Loki HTTP o gRPC endpoint standard
