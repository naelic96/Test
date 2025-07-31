import grpc
from concurrent import futures
import logging
import json
import time

from opentelemetry.proto.collector.logs.v1 import logs_service_pb2_grpc, logs_service_pb2
from opentelemetry.proto.logs.v1 import logs_pb2
from opentelemetry.proto.common.v1 import common_pb2
from opentelemetry.proto.resource.v1 import resource_pb2
from opentelemetry.exporter.otlp.proto.grpc.exporter import OTLPLogExporter

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("enricher")

# Solo questi verranno promossi a resource attributes (label per Loki)
ENRICH_KEYS_AS_LABELS = {"aws_region", "global_app"}

class OTLPLogExporterWrapper:
    def __init__(self, endpoint):
        self.exporter = OTLPLogExporter(endpoint=endpoint, insecure=True)

    def export(self, logs_data):
        return self.exporter.export(logs_data)

class EnrichServicer(logs_service_pb2_grpc.LogsServiceServicer):
    def __init__(self, enrich_path):
        self.enrich_path = enrich_path
        self.enrich_data = self.load_enrich_file()
        self.exporter = OTLPLogExporterWrapper("localhost:4317")
        logger.info(f"Loaded enrich data for {len(self.enrich_data)} targets.")

    def load_enrich_file(self):
        try:
            with open(self.enrich_path, "r") as f:
                raw = json.load(f)

            enrich_map = {}
            for entry in raw:
                targets = entry.get("Targets", [])
                labels = entry.get("labels", {})
                for target in targets:
                    enrich_map[target] = labels
            return enrich_map
        except Exception as e:
            logger.error(f"Errore nel caricamento del file JSON: {e}")
            return {}

    def Export(self, request, context):
        logger.info(f"📦 Ricevuti {len(request.resource_logs)} resource_logs")

        for resource_log in request.resource_logs:
            log_group_name = None
            resource_attrs = resource_log.resource.attributes

            for attr in resource_attrs:
                if attr.key == "cloudwatch_log_group_name":
                    log_group_name = attr.value.string_value
                    break

            logger.info(f"🔍 Trovato log_group_name: {log_group_name}")

            if not log_group_name or log_group_name not in self.enrich_data:
                continue

            enrich_tags = self.enrich_data[log_group_name]
            logger.info(f"✨ Enrichment: {enrich_tags}")

            for scope_log in resource_log.scope_logs:
                for log_record in scope_log.log_records:
                    # Aggiunge tutti gli enrich come attributi del log
                    for key, value in enrich_tags.items():
                        log_record.attributes.append(
                            common_pb2.KeyValue(key=key, value=common_pb2.AnyValue(string_value=value))
                        )

            # Promuove solo alcune chiavi a resource attributes (diventano label in Loki)
            for key, value in enrich_tags.items():
                if key in ENRICH_KEYS_AS_LABELS:
                    resource_attrs.append(
                        common_pb2.KeyValue(key=key, value=common_pb2.AnyValue(string_value=value))
                    )

        try:
            self.exporter.export(request)
            logger.info("✅ Logs inoltrati a Alloy su localhost:4317")
        except Exception as e:
            logger.error(f"❌ Errore invio a Alloy: {e}")

        return logs_service_pb2.ExportLogsServiceResponse()

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    logs_service_pb2_grpc.add_LogsServiceServicer_to_server(
        EnrichServicer("/tmp/enriched_tags.json"), server
    )
    server.add_insecure_port("[::]:50051")
    server.start()
    logger.info("🚀 Enricher gRPC server avviato su porta 50051")
    try:
        while True:
            time.sleep(3600)
    except KeyboardInterrupt:
        server.stop(0)

if __name__ == "__main__":
    serve()
