def match_log_group_to_resource(log_group_name):
    service = None
    parts = log_group_name.strip("/").split("/")
    if len(parts) >= 2 and parts[0] == "aws":
        service = parts[1]

    with lock:
        matches = []
        for arn, data in resource_cache.items():
            arn_parts = arn.split(":")
            resource_name = arn_parts[-1] if arn_parts else arn
            resource_service = arn_parts[2] if len(arn_parts) > 2 else None
            if log_group_name.endswith(resource_name) and (service is None or service == resource_service):
                matches.append({"arn": arn, "tags": data.get("tags", {})})

    if not matches:
        logging.warning(f"Telemetria: {log_group_name} → Nessun match trovato")
        return {}

    if len(matches) > 1:
        logging.info(f"Telemetria: {log_group_name} → Trovati {len(matches)} match multipli")

    enriched_tags = {}
    seen_values = {}

    for idx, match in enumerate(matches):
        base_tags = match["tags"].copy()
        base_tags["resource_arn"] = match["arn"]

        for k, v in base_tags.items():
            if k in ENRICH_KEYS_AS_LABELS:
                enriched_tags[k] = v
            else:
                if k not in seen_values:
                    enriched_tags[k] = v
                    seen_values[k] = v
                elif seen_values[k] != v:
                    arn_parts = match["arn"].split(":")
                    account_id = arn_parts[4] if len(arn_parts) > 4 else "unknown"
                    resource_part = normalize_suffix(arn_parts[5] if len(arn_parts) > 5 else "")
                    key = f"{resource_part}_{normalize_suffix(k)}"
                    enriched_tags[key] = v
                    enriched_tags[f"{resource_part}_account"] = account_id
                    logging.info(f"Tag duplicato '{k}' con valori diversi, aggiunto suffisso '{key}'")

    logging.info(f"Telemetria: {log_group_name} → tags arricchiti: {list(enriched_tags.keys())}")
    return enriched_tags
