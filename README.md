livedebugging {
  enabled = true
}

prometheus.exporter.cloudwatch "ec2_metrics" {
  sts_region = "eu-central-1"
  aws_sdk_version_v2 = true
  decoupled_scraping {
    enabled = true
  }
  discovery {
    type    = "AWS/EC2"
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

otelcol.storage.file "rds_persistent_state" {
  directory = "/var/lib/alloy/rds"
}

otelcol.storage.file "lambda_persistent_state" {
  directory = "/var/lib/alloy/lambda"
}

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
    logs    = [otelcol.processor.batch.aws_batch_before_grpc.input]
    traces  = [otelcol.connector.spanlogs.transform_trace_to_log.input, otelcol.processor.batch.batching_before_send.input]
    metrics = [otelcol.processor.batch.batching_before_send.input]
  }
}

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
    logs    = [otelcol.processor.batch.aws_batch_before_grpc.input]
    traces  = [otelcol.connector.spanlogs.transform_trace_to_log.input, otelcol.processor.batch.batching_before_send.input]
    metrics = [otelcol.processor.batch.batching_before_send.input]
  }
}

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

otelcol.receiver.otlp "alloy_receiver" {
  grpc {
    endpoint = "0.0.0.0:4317"
    max_recv_msg_size = "10000MiB"
  }
  output {
    logs    = [otelcol.processor.transform.set_custom_tag_as_attributes.input]
    traces  = [otelcol.connector.spanlogs.transform_trace_to_log.input, otelcol.processor.batch.batching_before_send.input]
    metrics = [otelcol.processor.batch.batching_before_send.input]
  }
}

otelcol.connector.spanlogs "transform_trace_to_log" {
  roots           = true
  spans           = true
  processes       = true
  events          = true
  span_attributes = ["http.method", "http.target"]
  event_attributes = ["log.severity", "log.message"]
  output {
    logs   = [otelcol.processor.transform.set_custom_tag_as_attributes.input]
    traces = [otelcol.connector.servicegraph.default.input, otelcol.exporter.otlp.export_to_tempo.input]
  }
}

otelcol.processor.transform "set_custom_tag_as_attributes" {
  error_mode = "ignore"
  log_statements {
    context = "log"
    statements = [
      `set(attributes["global_app"], attributes["global_app"])`,
    ]
  }
  output {
    logs = [otelcol.processor.batch.batching_before_send.input]
  }
}

otelcol.processor.batch "batching_before_send" {
  timeout = "10s"
  send_batch_size = 1000
  output {
    logs    = [otelcol.exporter.otlphttp.export_to_loki.input]
    metrics = [otelcol.exporter.prometheus.export_to_prometheus.input]
    traces  = [otelcol.connector.servicegraph.default.input, otelcol.exporter.otlp.export_to_tempo.input]
  }
}

otelcol.connector.servicegraph "default" {
  dimensions = ["http.method", "http.target"]
  output {
    metrics = [otelcol.exporter.prometheus.export_to_prometheus.input]
  }
}

otelcol.connector.spanmetrics "spanmetrics_from_traces" {
  dimensions = ["http.method", "http.target"]
  output {
    metrics = [otelcol.exporter.prometheus.export_to_prometheus.input]
  }
}

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
