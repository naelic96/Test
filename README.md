import grpc
import json
import logging
from concurrent import futures

from opentelemetry.proto.collector.logs.v1 import logs_service_pb2_grpc, logs_service_pb2
from opentelemetry.proto.logs.v1 import logs_pb2
from opentelemetry.proto.resource.v1 import resource_pb2

from opentelemetry.exporter.otlp.proto.grpc.logs_exporter import OTLPLogExporter
from opentelemetry.proto.collector.logs.v1.logs_service_pb2 import ExportLogsServiceResponse

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("enricher")

JSON_PATH = "/tmp/enriched_tags.json"
ALLOY_OTLP_ENDPOINT = "localhost:4317"

# Carica JSON arricchimenti una volta all'avvio
def load_enrichments():
    try:
        with open(JSON_PATH, "r") as f:
            data = json.load(f)
            # data è lista di dict con 'targets' (lista) e 'labels' (dict)
            logger.info(f"Loaded enrichment rules from {JSON_PATH}")
            return data
    except Exception as e:
        logger.error(f"Errore caricamento file JSON: {e}")
        return []

ENRICHMENTS = load_enrichments()

# Exporter OTLP verso Alloy
exporter = OTLPLogExporter(endpoint=ALLOY_OTLP_ENDPOINT, insecure=True)

class LogEnricherServicer(logs_service_pb2_grpc.LogsServiceServicer):
    def Export(self, request, context):
        logger.info(f"Received Export request with {len(request.resource_logs)} ResourceLogs")

        # Itera ResourceLogs
        for resource_log in request.resource_logs:
            # Recupera il valore del resource attribute 'cloudwatch_log_group_name'
            cloudwatch_name = None
            for attr in resource_log.resource.attributes:
                if attr.key == "cloudwatch_log_group_name":
                    cloudwatch_name = attr.value.string_value
                    break

            logger.info(f"cloudwatch_log_group_name: {cloudwatch_name}")

            if not cloudwatch_name:
                logger.info("Nessun cloudwatch_log_group_name trovato, skip enrichment per questo ResourceLog")
                continue

            # Cerca matching nel JSON (targets list)
            matched_labels = None
            for rule in ENRICHMENTS:
                targets = rule.get("targets", [])
                if cloudwatch_name in targets:
                    matched_labels = rule.get("labels", {})
                    logger.info(f"Match trovato per {cloudwatch_name}, labels: {matched_labels}")
                    break

            if not matched_labels:
                logger.info(f"Nessun matching labels trovato per {cloudwatch_name}")
                continue

            # Aggiungi le labels al resource.attributes (senza duplicati)
            existing_keys = {attr.key for attr in resource_log.resource.attributes}
            for k, v in matched_labels.items():
                if k not in existing_keys:
                    attr = resource_pb2.KeyValue(
                        key=k,
                        value=resource_pb2.AnyValue(string_value=str(v))
                    )
                    resource_log.resource.attributes.append(attr)
                    logger.debug(f"Aggiunta label {k}: {v}")

        # Inoltra il request arricchito ad Alloy OTLP endpoint
        try:
            # exporter.export prende ExportLogsServiceRequest
            result = exporter.export(request)
            logger.info("Inoltro a Alloy OTLP riuscito")
        except Exception as e:
            logger.error(f"Errore invio a Alloy: {e}")

        return ExportLogsServiceResponse()

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    logs_service_pb2_grpc.add_LogsServiceServicer_to_server(LogEnricherServicer(), server)
    server.add_insecure_port("[::]:50051")
    server.start()
    logger.info("Server gRPC Log Enricher in ascolto sulla porta 50051")
    server.wait_for_termination()

if __name__ == "__main__":
    serve()
