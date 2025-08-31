otelcol.connector.spanmetrics "generate_metrics" {
  dimensions = ["http.method", "http.target", "service.name"]
  output {
    metrics = [otelcol.exporter.prometheus.export_to_prometheus.input]
    traces  = [otelcol.exporter.otlp.export_to_tempo.input]
  }
}
