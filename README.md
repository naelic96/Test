prometheus.exporter.cloudwatch "ec2_metrics" {
  sts_region = "eu-central-1"
  aws_sdk_version_v2 = true
  decoupled_scraping {
    enabled = true
  }
  discovery {
    type = "AWS/EC2"
    regions = ["eu-central-1"]
    metric {
      name       = "CPUUtilization"
      statistics = ["Average"]
      period     = "5m"
    }
    metric {
      name       = "NetworkPacketsIn"
      statistics = ["Average"]
      period     = "5m"
    }
  }
}

prometheus.scrape "scrape_metrics" {
  targets         = prometheus.exporter.cloudwatch.ec2_metrics.targets
  forward_to      = [prometheus.remote_write.prometheus_writer.receiver]
  scrape_interval = "10s"
}

prometheus.remote_write "prometheus_writer" {
  endpoint {
    url = "http://prometheus.monitoring-preprod-axa-it.svc.cluster.local:9090/api/v1/write"
    tls_config {
      insecure_skip_verify = true
    }
  }
}

# =======================
# Persistent states
# =======================
otelcol.storage.file "rds_persistent_state" {
  directory = "/var/lib/alloy/rds"
}

otelcol.storage.file "lambda_persistent_state" {
  directory = "/var/lib/alloy/lambda"
}

# =======================
# CloudWatch RDS logs
# =======================
otelcol.receiver.awscloudwatch "cloudwatch_logs_rds" {
  region  = "eu-central-1"
  storage = otelcol.storage.file.rds_persistent_state.handler
  logs {
    poll_interval = "1m"
    max_events_per_request = 1000
    groups {
      autodiscover {
        limit  = 100
        prefix = "/aws/rds/"
      }
    }
  }
  output {
    logs    = [otelcol.processor.transform.inject_trace_info.input]
    traces  = [otelcol.processor.spanmetrics.gen_spanmetrics.input]
    metrics = [otelcol.processor.spanmetrics.gen_spanmetrics.input]
  }
}

# =======================
# CloudWatch Lambda logs
# =======================
otelcol.receiver.awscloudwatch "cloudwatch_logs_lambda" {
  region  = "eu-central-1"
  storage = otelcol.storage.file.lambda_persistent_state.handler
  logs {
    poll_interval = "1m"
    max_events_per_request = 1000
    start_from = "2025-08-01T01:00:00Z"
    groups {
      autodiscover {
        limit  = 100
        prefix = "/aws/lambda/"
      }
    }
  }
  output {
    logs    = [otelcol.processor.transform.inject_trace_info.input]
    traces  = [otelcol.processor.spanmetrics.gen_spanmetrics.input]
    metrics = [otelcol.processor.spanmetrics.gen_spanmetrics.input]
  }
}

# =======================
# Batch Processor
# =======================
otelcol.processor.batch "aws_batch_before_grpc" {
  timeout = "10s"
  send_batch_size = 500
  output {
    logs = [otelcol.exporter.otlp.to_python_grpc_server.input]
  }
}

otelcol.exporter.otlp "to_python_grpc_server" {
  client {
    endpoint = "localhost:50051"
    tls {
      insecure = true
      insecure_skip_verify = true
    }
  }
}

# =======================
# OTLP Receiver (application telemetry)
# =======================
otelcol.receiver.otlp "alloy_receiver" {
  grpc {
    endpoint = "0.0.0.0:4317"
    max_recv_msg_size = "10000MiB"
  }
  output {
    logs    = [otelcol.processor.transform.inject_trace_info.input]
    traces  = [otelcol.processor.spanmetrics.gen_spanmetrics.input]
    metrics = [otelcol.processor.spanmetrics.gen_spanmetrics.input]
  }
}

# =======================
# Span -> Log transformer
# =======================
otelcol.connector.spanlogs "transform_trace_to_log" {
  roots          = true
  spans          = true
  processes      = true
  events         = true
  span_attributes  = ["http.method", "http.target"]
  event_attributes = ["log.severity", "log.message"]
  output {
    logs   = [otelcol.processor.transform.inject_trace_info.input]
    traces = [otelcol.processor.spanmetrics.gen_spanmetrics.input]
  }
}

# =======================
# Inject trace_id/span_id into logs
# =======================
otelcol.processor.transform "inject_trace_info" {
  error_mode = "ignore"
  log_statements {
    context = "log"
    statements = [
      "set(attributes[\"trace_id\"], trace_id)",
      "set(attributes[\"span_id\"], span_id)",
    ]
  }
  output {
    logs = [otelcol.processor.batch.batching_before_send.input]
  }
}

# =======================
# SpanMetrics processor
# =======================
otelcol.processor.spanmetrics "gen_spanmetrics" {
  metrics_exporter = otelcol.exporter.prometheus.export_to_prometheus.input
  dimensions = ["http.method", "http.target", "service.name"]
  histogram {
    explicit_buckets = [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
  }
  output {
    traces = [otelcol.connector.servicegraph.default.input, otelcol.exporter.otlp.export_to_tempo.input]
  }
}

# =======================
# Batch before export
# =======================
otelcol.processor.batch "batching_before_send" {
  timeout = "10s"
  send_batch_size = 1000
  output {
    logs    = [otelcol.exporter.otlphttp.export_to_loki.input]
    metrics = [otelcol.exporter.prometheus.export_to_prometheus.input]
    traces  = [otelcol.connector.servicegraph.default.input, otelcol.exporter.otlp.export_to_tempo.input]
  }
}

# =======================
# Service graph connector
# =======================
otelcol.connector.servicegraph "default" {
  dimensions = ["http.method", "http.target"]
  output {
    metrics = [otelcol.exporter.prometheus.export_to_prometheus.input]
  }
}

# =======================
# Exporters
# =======================
otelcol.exporter.otlphttp "export_to_loki" {
  client {
    endpoint = "http://loki.monitoring-preprod-axa-it.svc.cluster.local:3100/otlp"
    tls {
      insecure = true
      insecure_skip_verify = true
    }
  }
}

otelcol.exporter.otlp "export_to_tempo" {
  client {
    endpoint = "http://tempo.monitoring-preprod-axa-it.svc.cluster.local:4317"
    tls {
      insecure = true
      insecure_skip_verify = true
    }
  }
}

otelcol.exporter.prometheus "export_to_prometheus" {
  forward_to = [prometheus.remote_write.prometheus_writer.receiver]
}
