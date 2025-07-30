import grpc
from concurrent import futures
import logging
import json

from opentelemetry.proto.collector.logs.v1 import logs_service_pb2_grpc, logs_service_pb2
from opentelemetry.proto.logs.v1 import logs_pb2
from opentelemetry.proto.resource.v1 import resource_pb2
from opentelemetry.proto.common.v1 import common_pb2

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("enricher")

ENRICH_FILE = "/tmp/enriched_tags.json"
LABEL_KEYS = {"aws_region", "global_app"}


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
            logger.info(f"🔍 log_group: {log_group}")

            if not log_group:
                logger.warning("⚠️ Nessun cloudwatch_log_group_name nei resource attributes")
                continue

            enrichment = enrichment_cache.get(log_group)
            if not enrichment:
                logger.warning(f"❌ Nessun enrichment trovato per {log_group}")
                continue

            logger.info(f"✅ Enrichment trovato: {enrichment}")

            # Aggiungi i campi arricchiti ai resource attributes
            for k, v in enrichment.items():
                if k == "targets":
                    continue
                resource_log.resource.attributes.append(
                    common_pb2.KeyValue(
                        key=k,
                        value=common_pb2.AnyValue(string_value=v)
                    )
                )

        # Inoltro a Alloy su 4317
        try:
            with grpc.insecure_channel("localhost:4317") as channel:
                stub = logs_service_pb2_grpc.LogsServiceStub(channel)
                stub.Export(request)
                logger.info("🚀 Log arricchiti inoltrati ad Alloy")
        except Exception as e:
            logger.error(f"❌ Errore invio a Alloy: {e}")

        return logs_service_pb2.ExportLogsServiceResponse()


def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    logs_service_pb2_grpc.add_LogsServiceServicer_to_server(EnricherService(), server)
    server.add_insecure_port("[::]:50051")
    logger.info("🚀 Enricher gRPC Server in ascolto su :50051")
    server.start()
    server.wait_for_termination()


if __name__ == "__main__":
    serve()
