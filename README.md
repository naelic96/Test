def get_boto3_client(
    service: str,
    region_name: Optional[str] = None,
    role_arn: Optional[str] = None,
    role_session_name: str = "EnricherSession",
):
    """
    Usa la default provider chain per le credenziali sorgente (ENV -> IRSA/WebIdentity -> profili -> Instance Role).
    Se role_arn è valorizzato, assume il ruolo target con STS (supporta EXTERNAL_ID e DURATION_SECONDS).
    """
    import boto3, botocore, os

    # 1) Sessione di base (sorgente): NON forziamo env parziali -> boto3 sceglie automaticamente la credenziale giusta.
    base_session = boto3.session.Session(region_name=region_name)

    # 2) Se non devi assumere, restituisci il client diretto
    if not role_arn:
        return base_session.client(service)

    # 3) Parametri per AssumeRole
    external_id = os.getenv("EXTERNAL_ID")  # opzionale
    duration = int(os.getenv("DURATION_SECONDS", "3600"))  # opzionale (rispetta i limiti del ruolo/policy)
    sts_cfg = botocore.config.Config(
        retries={"max_attempts": 5, "mode": "standard"},
        # Evita redirect globali lenti
        s3={"addressing_style": "virtual"},
    )

    sts = base_session.client("sts", config=sts_cfg)
    assume_kwargs = {
        "RoleArn": role_arn,
        "RoleSessionName": role_session_name,
        "DurationSeconds": duration,
    }
    if external_id:
        assume_kwargs["ExternalId"] = external_id

    try:
        resp = sts.assume_role(**assume_kwargs)
        creds = resp["Credentials"]
    except Exception as e:
        # Fallimento esplicito: meglio sapere subito perché (trust policy, external id, token scaduto, etc.)
        logger.error("AssumeRole failed", extra={
            "role_arn": role_arn,
            "external_id": bool(external_id),
            "error": str(e),
        })
        raise

    # 4) Client del servizio con le credenziali ASSUNTE
    return boto3.client(
        service,
        region_name=region_name or base_session.region_name,
        aws_access_key_id=creds["AccessKeyId"],
        aws_secret_access_key=creds["SecretAccessKey"],
        aws_session_token=creds["SessionToken"],
    )



    cfg_client = get_boto3_client(
    "config",
    region_name=REGION_FILTER or None,
    role_arn=ASSUME_ROLE_ARN,            # <— usa il ruolo target
    role_session_name=ROLE_SESSION_NAME, # es. "GrpcEnricherSession"
)
