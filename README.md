import grpc
from concurrent import futures
import logging
import json
import time

from opentelemetry.proto.collector.logs.v1 import logs_service_pb2_grpc, logs_service_pb2
from opentelemetry.proto.logs.v1 import logs_pb2
from opentelemetry.proto.common.v1 import common_pb2

from opentelemetry.proto.resource.v1 import resource_pb2
from google.protobuf import json_format

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("enricher")

# Lista di campi che diventeranno label Loki
ENRICH_KEYS_AS_LABELS = {"aws_region", "global_app"}

class EnrichServicer(logs_service_pb2_grpc.LogsServiceServicer):
    def __init__(self, enrich_path):
        self.enrich_path = enrich_path
        self.enrich_data = self.load_enrich_file()
        self.channel = grpc.insecure_channel("localhost:4317")
        self.stub = logs_service_pb2_grpc.LogsServiceStub(self.channel)
        logger.info(f"Loaded enrich data for targets: {list(self.enrich_data.keys())}")

    def load_enrich_file(self):
        try:
            with open(self.enrich_path, "r") as f:
                raw = json.load(f)
            mapping = {}
            for entry in raw:
                targets = entry.get("targets", [])
                labels = entry.get("labels", {})
                for target in targets:
                    mapping[target] = labels
            return mapping
        except Exception as e:
            logger.error(f"Error loading enrich file: {e}")
            return {}

    def Export(self, request, context):
        logger.info(f"Received Export request with {len(request.resource_logs)} resource_logs")

        for resource_log in request.resource_logs:
            resource_attrs = {kv.key: kv.value.string_value for kv in resource_log.resource.attributes}
            log_group_name = resource_attrs.get("cloudwatch_log_group_name")

            logger.info(f"cloudwatch_log_group_name: {log_group_name}")

            if not log_group_name or log_group_name not in self.enrich_data:
                continue

            enrich_labels = self.enrich_data[log_group_name]
            logger.info(f"Enriching logs for {log_group_name} with {enrich_labels}")

            # Enrich all logs inside this resource_log
            for scope_log in resource_log.scope_logs:
                for log_record in scope_log.log_records:
                    for key, val in enrich_labels.items():
                        any_val = common_pb2.AnyValue(string_value=val)
                        log_record.attributes.append(common_pb2.KeyValue(key=key, value=any_val))

        try:
            self.stub.Export(request)
            logger.info("✅ Enriched logs sent to Alloy at :4317")
        except Exception as e:
            logger.error(f"❌ Error sending to Alloy: {e}")

        return logs_service_pb2.ExportLogsServiceResponse()

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    logs_service_pb2_grpc.add_LogsServiceServicer_to_server(
        EnrichServicer("/tmp/enriched_tags.json"),
        server
    )
    server.add_insecure_port("[::]:50051")
    server.start()
    logger.info("🚀 gRPC Enricher Server started on port 50051")
    try:
        while True:
            time.sleep(3600)
    except KeyboardInterrupt:
        server.stop(0)

if __name__ == "__main__":
    serve()
