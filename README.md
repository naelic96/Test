import grpc
from concurrent import futures
import logging
import json
import time

from opentelemetry.proto.collector.logs.v1 import logs_service_pb2_grpc, logs_service_pb2
from opentelemetry.proto.common.v1 import common_pb2
from opentelemetry.proto.logs.v1 import logs_pb2

# Configurazione del logger
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("enricher")

ENRICH_KEYS_AS_LABELS = {"aws_region", "global_app"}

# Classe custom per il client gRPC che invia i log arricchiti
class OTLPLogExporterCustom:
    def __init__(self, endpoint):
        self.channel = grpc.insecure_channel(endpoint)
        self.stub = logs_service_pb2_grpc.LogsServiceStub(self.channel)

    def export(self, logs_data):
        try:
            response = self.stub.Export(logs_data)
            logger.info("Logs forwarded successfully to Alloy")
            return response
        except grpc.RpcError as e:
            logger.error(f"Error forwarding logs: {e}")
            return None

# Classe del servizio gRPC che arricchisce e inoltra i log
class EnrichServicer(logs_service_pb2_grpc.LogsServiceServicer):
    def __init__(self, enrich_path):
        self.enrich_path = enrich_path
        self.enrich_data = self.load_enrich_file()
        self.exporter = OTLPLogExporterCustom("localhost:4317")
        logger.info(f"Loaded enrich data for targets: {list(self.enrich_data.keys())}")

    def load_enrich_file(self):
        try:
            with open(self.enrich_path, "r") as f:
                data = json.load(f)
            return data
        except Exception as e:
            logger.error(f"Error loading enrich file: {e}")
            return {}

    def Export(self, request, context):
        logger.info(f"Received Export request with {len(request.resource_logs)} resource_logs")

        # Itera sui log ricevuti
        for resource_log in request.resource_logs:
            resource_attrs = resource_log.resource.attributes

            log_group_name = None
            for k, v in resource_attrs.items():
                if k == "cloudwatch_log_group_name":
                    log_group_name = v.string_value
                    break

            logger.info(f"log_group_name: {log_group_name}")

            # Se non esiste un log_group_name valido, salta
            if not log_group_name or log_group_name not in self.enrich_data:
                continue

            enrich_labels = self.enrich_data[log_group_name]
            logger.info(f"Enriching logs for {log_group_name} with {enrich_labels}")

            # Arricchisci i log
            for scope_log in resource_log.scope_logs:
                for log_record in scope_log.logs:
                    # Promuovi alcuni campi come label, altri restano come attributi
                    for key, val in enrich_labels.items():
                        if key in ENRICH_KEYS_AS_LABELS:
                            # Aggiungi come label
                            log_record.labels[key] = val
                        else:
                            # Aggiungi come attributo
                            log_record.attributes[key].CopyFrom(common_pb2.AnyValue(string_value=val))

        # Inoltra i log arricchiti a Alloy tramite gRPC
        try:
            self.exporter.export(request)
            logger.info("Forwarded enriched logs to Alloy")
        except Exception as e:
            logger.error(f"Error forwarding logs: {e}")

        return logs_service_pb2.ExportLogsServiceResponse()

# Funzione che avvia il server gRPC
def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    logs_service_pb2_grpc.add_LogsServiceServicer_to_server(
        EnrichServicer("/tmp/enriched_tags.json"),
        server
    )
    server.add_insecure_port("[::]:50051")
    server.start()
    logger.info("gRPC Enricher Server started on port 50051")
    try:
        while True:
            time.sleep(3600)
    except KeyboardInterrupt:
        server.stop(0)

if __name__ == "__main__":
    serve()
