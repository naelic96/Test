import json
import logging
import grpc
from concurrent import futures
from threading import Lock

from opentelemetry.proto.collector.logs.v1 import logs_service_pb2_grpc, logs_service_pb2
from opentelemetry.proto.logs.v1 import logs_pb2
from opentelemetry.proto.resource.v1 import resource_pb2

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("enricher")

ENRICH_FILE_PATH = "/tmp/enriched_tags.json"
FORWARD_TO_ALLOY_ENDPOINT = "localhost:4317"
LABELS_TO_PROMOTE = ["aws_region", "global_app"]  # solo queste diventano label, le altre restano campi

class LogEnricherServicer(logs_service_pb2_grpc.LogsServiceServicer):
    def __init__(self, enrich_file_path, labels_to_promote):
        self.enrich_file_path = enrich_file_path
        self.labels_to_promote = labels_to_promote
        self.lock = Lock()
        self.enrich_data = self.load_enrich_file()

        # gRPC client per inoltro a Alloy
        self.channel = grpc.insecure_channel(FORWARD_TO_ALLOY_ENDPOINT)
        self.client = logs_service_pb2_grpc.LogsServiceStub(self.channel)

    def load_enrich_file(self):
        try:
            with open(self.enrich_file_path, "r") as f:
                data = json.load(f)
                logger.info(f"Enrichment file caricato: {self.enrich_file_path}")
                return data
        except Exception as e:
            logger.error(f"Errore durante il caricamento del file JSON: {e}")
            return []

    def match_enrich_labels(self, log_group_name):
        # Match esatto tra log_group_name e uno dei targets nell'enrich_data
        for entry in self.enrich_data:
            targets = entry.get("targets", [])
            if log_group_name in targets:
                return entry.get("labels", {})
        return {}

    def enrich_log_record(self, log_record, enrich_labels):
        # Enrich attributes con labels e campi, promuovendo alcune a labels
        # log_record è LogRecord protobuf, ha attributes (repeated KeyValue)

        # Convert attributes to dict
        attr_dict = {kv.key: kv.value.string_value for kv in log_record.attributes}

        # Add enrich labels as fields (campi)
        for k, v in enrich_labels.items():
            attr_dict[k] = v

        # Ricrea la lista attributes e separa labels da campi
        new_attributes = []
        new_labels = {}

        for k, v in attr_dict.items():
            if k in self.labels_to_promote:
                new_labels[k] = v
            else:
                new_attributes.append(logs_pb2.KeyValue(key=k, value=logs_pb2.AnyValue(string_value=v)))

        # Sostituisci attributi con campi aggiornati
        log_record.attributes[:] = new_attributes

        # Le labels devono andare nel Resource attributes della ResourceLogs
        # Lo aggiustiamo in Export()

        return new_labels  # ritorna labels promosse per Resource

    def Export(self, request, context):
        logger.info(f"Ricevuta Export chiamata con {len(request.resource_logs)} ResourceLogs")

        # Ricarica il file ogni volta (potresti mettere cache con refresh a intervalli)
        with self.lock:
            self.enrich_data = self.load_enrich_file()

        # Itera su ResourceLogs
        for resource_log in request.resource_logs:
            # Ottieni il log_group_name da Resource attributes
            log_group_name = None
            for attr in resource_log.resource.attributes:
                if attr.key == "cloudwatch_log_group_name":
                    log_group_name = attr.value.string_value
                    break

            if not log_group_name:
                logger.warning("ResourceLogs senza cloudwatch_log_group_name, salto enrich")
                continue

            logger.info(f"Log group trovato: {log_group_name}")

            enrich_labels = self.match_enrich_labels(log_group_name)
            logger.info(f"Labels trovate per enrich: {enrich_labels}")

            # Enrich dei LogRecords nei ScopeLogs
            # Inoltre costruiamo nuovi attributi Resource per le labels promosse
            promoted_labels = {}

            for scope_log in resource_log.scope_logs:
                for log_record in scope_log.logs:
                    labels_from_log = self.enrich_log_record(log_record, enrich_labels)
                    promoted_labels.update(labels_from_log)

            # Aggiungi/aggiorna Resource attributes con labels promosse
            # Rimuovi quelle che eventualmente esistono con lo stesso nome prima di aggiungere
            resource_log.resource.attributes[:] = [
                attr for attr in resource_log.resource.attributes if attr.key not in promoted_labels
            ]
            for k, v in promoted_labels.items():
                resource_log.resource.attributes.append(
                    resource_pb2.KeyValue(key=k, value=resource_pb2.AnyValue(string_value=v))
                )

        # Inoltra i logs arricchiti a Alloy collector
        try:
            response = self.client.Export(request)
            logger.info("Log arricchiti inoltrati a Alloy con successo")
        except Exception as e:
            logger.error(f"Errore invio a Alloy: {e}")

        return logs_service_pb2.ExportLogsServiceResponse()

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    logs_service_pb2_grpc.add_LogsServiceServicer_to_server(
        LogEnricherServicer(ENRICH_FILE_PATH, LABELS_TO_PROMOTE), server
    )
    server.add_insecure_port("[::]:50051")
    server.start()
    logger.info("gRPC LogEnricher server in ascolto sulla porta 50051")
    server.wait_for_termination()

if __name__ == "__main__":
    serve()
