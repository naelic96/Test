import grpc
from concurrent import futures
import logging
import json
import os

from opentelemetry.proto.collector.logs.v1 import logs_service_pb2_grpc, logs_service_pb2
from opentelemetry.proto.logs.v1 import logs_pb2
from opentelemetry.proto.resource.v1 import resource_pb2
from opentelemetry.proto.common.v1 import common_pb2

from opentelemetry.exporter.otlp.proto.grpc.logs_exporter import OTLPLogExporter
from opentelemetry.sdk.logs import LogRecord, LoggerProvider
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk._logs.export import BatchLogProcessor
from opentelemetry.sdk._logs import set_logger_provider

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("enricher")

ENRICH_FILE = "/tmp/enriched_tags.json"
LABEL_KEYS = {"aws_region", "global_app"}

# Cache the enrichment data
def load_enrichment():
    try:
        with open(ENRICH_FILE, "r") as f:
            data = json.load(f)
            return {item["targets"]: item for item in data}
    except Exception as e:
        logger.error(f"Errore durante il caricamento del file JSON: {e}")
        return {}

enrichment_cache = load_enrichment()


class EnricherService(logs_service_pb2_grpc.LogsServiceServicer):
    def Export(self, request, context):
        logger.info("📥 Log ricevuto")

        for resource_log in request.resource_logs:
            resource_attrs = {
                attr.key: attr.value.string_value
                for attr in resource_log.resource.attributes
                if attr.value.WhichOneof("value") == "string_value"
            }

            log_group = resource_attrs.get("cloudwatch_log_group_name")
            if not log_group:
                logger.warning("⚠️ Nessun log_group trovato nei resource attributes")
                continue

            logger.info(f"🔎 log_group: {log_group}")

            enrichment = enrichment_cache.get(log_group)
            if not enrichment:
                logger.warning(f"❌ Nessun enrichment trovato per {log_group}")
                continue

            logger.info(f"✅ Enrichment trovato: {enrichment}")

            # Aggiunge enrichment ai resource attributes
            for key, value in enrichment.items():
                if key == "targets":
                    continue
                kv = common_pb2.KeyValue(
                    key=key,
                    value=common_pb2.AnyValue(string_value=value)
                )
                resource_log.resource.attributes.append(kv)

        # Inoltra i log a Alloy (4317)
        try:
            exporter = OTLPLogExporter(endpoint="localhost:4317", insecure=True)
            exporter.export([self._convert_proto_log(log) for resource_log in request.resource_logs for scope_log in resource_log.scope_logs for log in scope_log.log_records])
            logger.info("🚀 Log arricchiti inoltrati ad Alloy")
        except Exception as e:
            logger.error(f"❌ Errore invio a Alloy: {e}")

        return logs_service_pb2.ExportLogsServiceResponse()

    def _convert_proto_log(self, log_record_proto):
        # Convert protobuf log record to OpenTelemetry SDK LogRecord
        return LogRecord(
            timestamp=log_record_proto.time_unix_nano,
            severity_number=log_record_proto.severity_number,
            severity_text=log_record_proto.severity_text,
            body=log_record_proto.body.string_value if log_record_proto.body else "",
            attributes={attr.key: attr.value.string_value for attr in log_record_proto.attributes},
            resource=Resource.create({}),  # dummy; real resource is already included in request
            context=None
        )


def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    logs_service_pb2_grpc.add_LogsServiceServicer_to_server(EnricherService(), server)
    server.add_insecure_port("[::]:50051")
    logger.info("🚀 gRPC Enricher Server in ascolto sulla porta 50051...")
    server.start()
    server.wait_for_termination()


if __name__ == "__main__":
    serve()
