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
  chunk_idle_period: 10m         # tempo massimo prima di caricare chunk non usati
  max_chunk_age: 1h              # chunk rolling time
  chunk_target_size: 1048576     # 1MB chunk target size
  chunk_retain_period: 30s       # tempo chunk tenuti in memoria
  max_transfer_retries: 0

distributor:
  # Distributore per ricezione dati
  ring:
    kvstore:
      store: inmemory
    replication_factor: 1

ingester_client:
  # Config per il client ingester
  grpc_client_config:
    max_send_msg_size: 10485760  # 10MB
    max_recv_msg_size: 10485760  # 10MB

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
      # Cache su filesystem locale
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
  max_look_back_period: 336h  # 14 giorni

table_manager:
  retention_deletes_enabled: true
  retention_period: 720h  # 30 giorni

ruler:
  enable_api: true
  alertmanager_url: http://alertmanager:9093

# Config OTLP HTTP receiver per ricevere log OpenTelemetry
receivers:
  otlp:
    protocols:
      grpc:
        max_recv_msg_size_mib: 20  # 20 MiB
      http:
        max_recv_msg_size_mib: 20

prometheus:
  wal_directory: /loki/wal

# Promozione risorse OTLP a labels Loki (esempio)
label_configs:
  - source_labels: [__resource_aws_region]
    target_label: aws_region
  - source_labels: [__resource_global_app]
    target_label: global_app

# Disable memcache (non presente)
# cache_config: {}           resource_attrs = {kv.key: kv.value.string_value for kv in resource_log.resource.attributes}
            log_group_name = resource_attrs.get("cloudwatch_log_group_name")

            logger.info(f"cloudwatch_log_group_name: {log_group_name}")

            if not log_group_name or log_group_name not in self.enrich_data:
                continue

            enrich_labels = self.enrich_data[log_group_name]
            logger.info(f"Enriching logs for {log_group_name} with {enrich_labels}")

            # Enrich all logs inside this resource_log
            for scope_log in resource_log.scope_logs:
                for log_record in scope_log.log_records:
                    for key, val in enrich_labels.items():
                        any_val = common_pb2.AnyValue(string_value=val)
                        log_record.attributes.append(common_pb2.KeyValue(key=key, value=any_val))

        try:
            self.stub.Export(request)
            logger.info("✅ Enriched logs sent to Alloy at :4317")
        except Exception as e:
            logger.error(f"❌ Error sending to Alloy: {e}")

        return logs_service_pb2.ExportLogsServiceResponse()

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    logs_service_pb2_grpc.add_LogsServiceServicer_to_server(
        EnrichServicer("/tmp/enriched_tags.json"),
        server
    )
    server.add_insecure_port("[::]:50051")
    server.start()
    logger.info("🚀 gRPC Enricher Server started on port 50051")
    try:
        while True:
            time.sleep(3600)
    except KeyboardInterrupt:
        server.stop(0)

if __name__ == "__main__":
    serve()
