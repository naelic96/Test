import grpc
import json
import logging
from concurrent import futures

from opentelemetry.proto.collector.logs.v1 import logs_service_pb2_grpc, logs_service_pb2
from opentelemetry.proto.logs.v1 import logs_pb2
from opentelemetry.proto.resource.v1 import resource_pb2

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("enricher")

class LogEnricherServicer(logs_service_pb2_grpc.LogsServiceServicer):
    def __init__(self, enrich_config_file, label_keys_to_promote, alloy_target):
        self.enrich_config_file = enrich_config_file
        self.label_keys_to_promote = label_keys_to_promote
        self.alloy_target = alloy_target
        self.load_enrichments()

        # Setup channel & stub for forwarding to Alloy
        self.channel = grpc.insecure_channel(self.alloy_target)
        self.stub = logs_service_pb2_grpc.LogsServiceStub(self.channel)

    def load_enrichments(self):
        try:
            with open(self.enrich_config_file, "r") as f:
                self.enrich_data = json.load(f)
                logger.info(f"Enrichment config loaded from {self.enrich_config_file}")
        except Exception as e:
            logger.error(f"Errore durante il caricamento del file JSON: {e}")
            self.enrich_data = []

    def find_labels(self, log_group_name):
        for entry in self.enrich_data:
            targets = entry.get("Targets", [])
            if log_group_name in targets:
                return entry.get("Labels", {})
        return {}

    def Export(self, request, context):
        logger.info(f"Ricevuto Export con {len(request.resource_logs)} resource_logs")

        enriched_resource_logs = []

        for resource_log in request.resource_logs:
            resource = resource_log.resource
            log_group_name = None
            for attr in resource.attributes:
                if attr.key == "cloudwatch_log_group_name":
                    log_group_name = attr.value.string_value
                    break

            labels_to_add = {}
            if log_group_name:
                labels_to_add = self.find_labels(log_group_name)
                logger.info(f"Enriching logs per log_group_name='{log_group_name}': {labels_to_add}")

            new_attributes = list(resource.attributes)

            for k, v in labels_to_add.items():
                if k in self.label_keys_to_promote:
                    new_attributes.append(resource_pb2.KeyValue(
                        key=f"label_{k}",
                        value=resource_pb2.AnyValue(string_value=str(v))
                    ))
                else:
                    new_attributes.append(resource_pb2.KeyValue(
                        key=k,
                        value=resource_pb2.AnyValue(string_value=str(v))
                    ))

            new_resource = resource_pb2.Resource(attributes=new_attributes)

            enriched_resource_log = logs_pb2.ResourceLogs(
                resource=new_resource,
                scope_logs=resource_log.scope_logs,
            )
            enriched_resource_logs.append(enriched_resource_log)

        enriched_request = logs_service_pb2.ExportLogsServiceRequest(
            resource_logs=enriched_resource_logs
        )

        try:
            # Forward enriched logs to Alloy
            resp = self.stub.Export(enriched_request)
            logger.info("Inoltro a Alloy avvenuto con successo")
        except Exception as e:
            logger.error(f"Errore invio a Alloy: {e}")

        return logs_service_pb2.ExportLogsServiceResponse()

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    enrich_config_file = "/tmp/enriched_tags.json"
    label_keys_to_promote = ["aws_region", "global_app"]
    alloy_target = "localhost:4317"  # Cambia se Alloy è altrove

    logs_service_pb2_grpc.add_LogsServiceServicer_to_server(
        LogEnricherServicer(enrich_config_file, label_keys_to_promote, alloy_target), server
    )
    server.add_insecure_port('[::]:50051')
    logger.info("Starting gRPC server on port 50051...")
    server.start()
    server.wait_for_termination()

if __name__ == "__main__":
    serve()
