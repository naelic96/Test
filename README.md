import grpc
from concurrent import futures
import logging
import json
import time

from opentelemetry.proto.collector.logs.v1 import logs_service_pb2_grpc, logs_service_pb2
from opentelemetry.proto.logs.v1 import logs_pb2
from opentelemetry.proto.common.v1 import common_pb2

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("enricher")

class OTLPLogExporterCustom:
    def __init__(self, endpoint):
        try:
            from opentelemetry.exporter.otlp.proto.grpc.logs_exporter import OTLPLogExporter
            self._exporter = OTLPLogExporter(endpoint=endpoint, insecure=True)
        except ImportError as e:
            logger.error(f"❌ OTLPLogExporter not available: {e}")
            raise

    def export(self, logs_data):
        return self._exporter.export(logs_data)

class EnrichServicer(logs_service_pb2_grpc.LogsServiceServicer):
    def __init__(self, enrich_path):
        self.enrich_path = enrich_path
        self.enrich_data = self.load_enrich_file()
        self.exporter = OTLPLogExporterCustom("localhost:4317")
        logger.info(f"✅ Enrich data loaded for targets: {list(self.enrich_data.keys())}")

    def load_enrich_file(self):
        try:
            with open(self.enrich_path, "r") as f:
                data = json.load(f)
            assert isinstance(data, dict), "Enrich file must be a dictionary"
            return data
        except Exception as e:
            logger.error(f"❌ Failed to load enrich file: {e}")
            return {}

    def Export(self, request, context):
        logger.info(f"📥 Received Export request with {len(request.resource_logs)} resource_logs")

        for resource_log in request.resource_logs:
            log_group_name = None
            for attr in resource_log.resource.attributes:
                if attr.key == "cloudwatch_log_group_name":
                    log_group_name = attr.value.string_value
                    break

            if not log_group_name:
                logger.warning("⚠️ Missing 'cloudwatch_log_group_name' in resource attributes")
                continue

            logger.info(f"🔍 log_group_name: {log_group_name}")

            if log_group_name not in self.enrich_data:
                logger.info(f"ℹ️ No enrich rules found for: {log_group_name}")
                continue

            enrich_fields = self.enrich_data[log_group_name]
            logger.info(f"🛠️ Enriching logs for {log_group_name} with {enrich_fields}")

            for scope_log in resource_log.scope_logs:
                for log_record in scope_log.log_records:
                    for key, value in enrich_fields.items():
                        log_record.attributes[key] = common_pb2.AnyValue(string_value=value)

        try:
            self.exporter.export(request)
            logger.info("✅ Logs forwarded to Alloy (4317)")
        except Exception as e:
            logger.error(f"❌ Error forwarding logs to Alloy: {e}")

        return logs_service_pb2.ExportLogsServiceResponse()

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    logs_service_pb2_grpc.add_LogsServiceServicer_to_server(
        EnrichServicer("/tmp/enriched_tags.json"),
        server
    )
    server.add_insecure_port("[::]:50051")
    server.start()
    logger.info("🚀 Enricher gRPC server listening on port 50051")
    try:
        while True:
            time.sleep(3600)
    except KeyboardInterrupt:
        server.stop(0)
        logger.info("🛑 Server stopped")

if __name__ == "__main__":
    serve()
