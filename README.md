
import json
import logging
import grpc
from concurrent import futures
import time

from opentelemetry.proto.collector.logs.v1 import logs_service_pb2_grpc, logs_service_pb2
from opentelemetry.proto.logs.v1 import logs_pb2
from opentelemetry.exporter.otlp.proto.grpc.logs_exporter import OTLPLogExporter
from opentelemetry.proto.common.v1 import common_pb2

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("enricher")

class LogEnricherServicer(logs_service_pb2_grpc.LogsServiceServicer):
    def __init__(self, enrich_file_path, labels_to_promote=None):
        self.enrich_file_path = enrich_file_path
        self.enrich_data = self.load_enrich_file()
        self.labels_to_promote = labels_to_promote or []  # labels da promuovere in labels Loki
        self.exporter = OTLPLogExporter(endpoint="localhost:4317", insecure=True)

    def load_enrich_file(self):
        try:
            with open(self.enrich_file_path, "r") as f:
                data = json.load(f)
                logger.info(f"Enrich file loaded: {self.enrich_file_path}")
                return data
        except Exception as e:
            logger.error(f"Errore durante il caricamento del file JSON: {e}")
            return []

    def Export(self, request, context):
        logger.info(f"Ricevuti {len(request.resource_logs)} resource_logs")

        for resource_log in request.resource_logs:
            # Estraggo cloudwatch_log_group_name da resource attributes
            log_group_name = None
            for attr in resource_log.resource.attributes:
                if attr.key == "cloudwatch_log_group_name":
                    log_group_name = attr.value.string_value
                    break
            logger.info(f"Resource cloudwatch_log_group_name: {log_group_name}")

            if not log_group_name:
                logger.warning("Nessun cloudwatch_log_group_name trovato nel resource attributes, salto")
                continue

            # Cerco matching nei targets caricati da JSON
            matching_entry = None
            for entry in self.enrich_data:
                targets = entry.get("Targets", [])
                if log_group_name in targets:
                    matching_entry = entry
                    break

            if not matching_entry:
                logger.info(f"Nessun matching trovato per log_group_name: {log_group_name}")
                continue

            logger.info(f"Matching trovato per log_group_name: {log_group_name}, labels da aggiungere: {matching_entry.get('Labels',{})}")

            # Per ogni log record, arricchisco le labels e preparo la nuova struttura
            for scope_log in resource_log.scope_logs:
                for log_record in scope_log.logs:
                    # Stampo il corpo log per debug
                    logger.info(f"Log body prima enrich: {log_record.body}")

                    # Aggiungo labels come attributi (labels per Loki sono resource attributes o attributes)
                    # Promuovo solo quelli in labels_to_promote come labels vere, gli altri come attributi normali
                    new_attributes = list(log_record.attributes)

                    # Trasformo Labels da matching_entry in attributi e labels selezionate in separate
                    # In OpenTelemetry logs le "labels" sono generalmente resource attributes o log attributes
                    # Qui aggiungiamo come attributi (key-value)
                    for k, v in matching_entry.get("Labels", {}).items():
                        # Se la label è da promuovere, aggiungo prefisso "loki_label_" per distinguerle (o altra logica)
                        if k in self.labels_to_promote:
                            # Se vuoi puoi usare un attributo speciale o lasciare così (dipende da ingest su Loki)
                            new_attributes.append(common_pb2.KeyValue(key=k, value=common_pb2.AnyValue(string_value=v)))
                        else:
                            # Campi normali come attributi
                            new_attributes.append(common_pb2.KeyValue(key=k, value=common_pb2.AnyValue(string_value=v)))

                    # Sovrascrivo gli attributi del log_record
                    log_record.attributes[:] = new_attributes

                    # Stampo il corpo arricchito per debug
                    logger.info(f"Log body dopo enrich: {log_record.body}")
                    logger.info(f"Attributi arricchiti: {[f'{a.key}={a.value.string_value}' for a in log_record.attributes]}")

        # Inoltro tutto a Alloy OTLP (localhost:4317)
        try:
            self.exporter.export(request)
            logger.info("Log arricchiti inviati a Alloy con successo")
        except Exception as e:
            logger.error(f"Errore invio a Alloy: {e}")

        return logs_service_pb2.ExportLogsServiceResponse()

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    logs_service_pb2_grpc.add_LogsServiceServicer_to_server(
        LogEnricherServicer("/tmp/enriched_tags.json", labels_to_promote=["aws_region", "global_app"]),
        server
    )
    server.add_insecure_port("[::]:50051")
    logger.info("Server grpc LogEnricher in ascolto sulla porta 50051...")
    server.start()
    try:
        while True:
            time.sleep(60*60*24)
    except KeyboardInterrupt:
        server.stop(0)

if __name__ == "__main__":
    serve()
