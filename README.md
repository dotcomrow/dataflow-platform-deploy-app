# dataflow-platform-deploy-app

Production NiFi flow app for platform application deployment orchestration.

Flow ownership:

1. Consume prepared deployment requests from `batch.platform.deploy.prepared.v1`.
2. Extract the platform app deployment operation metadata.
3. Mark the operation as running through `platform-deploy-service`.
4. Provide the NiFi process group where the long-running deployment orchestration steps will be added.
5. Route orchestration failures to `batch.platform.deploy.prepared.dlq.v1`.

The app references the shared `dataflow/nifi-external` NiFi cluster but does
not own that cluster or its TLS auth secret.

The NiFi Registry flow ID is manifest-owned:

- `46d4e1b6-e423-49c2-9b5f-636e9f12f171`

The `platform-deploy-nifi-registry-bootstrap` Sync hook creates that Registry
flow and version `1` if they do not already exist.

Required Vault values before first production sync:

- `secret/data/k8s-kafka-nifi-registry-bucket-id#value`
- `secret/data/kafka-nifi-username#value`
- `secret/data/kafka-nifi-password#value`
- `secret/data/platform-deploy-service#token`

This project defines and configures the NiFi orchestration handoff. The platform
deploy service submits the preparation job to Flink, and that Flink job publishes
prepared deployment requests for this flow to consume. The Terraform/Cloudflare
execution processors still need to be added to this flow.
