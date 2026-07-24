# dataflow-platform-deploy-app

Production NiFi flow app for platform application deployment orchestration.

Flow ownership:

1. Consume prepared deployment requests from `batch.platform.deploy.prepared.v1`.
2. Extract the platform app deployment operation metadata.
3. Mark the operation as running through `platform-deploy-service`.
4. Block executor calls unless `execution_enabled` is set to `"true"`.
5. Route create operations through prod deploy first, then preview deploy.
6. Route update/redeploy operations through preview destroy, prod destroy,
   prod deploy, then preview deploy.
7. Route destroy operations through preview destroy first, then prod destroy.
8. Call back to `platform-deploy-service` to finish the operation.
9. Route orchestration failures to `batch.platform.deploy.prepared.dlq.v1`.

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
prepared deployment requests for this flow to consume.

Execution boundary:

- NiFi owns the visible workflow, sequencing, retries, and failure routing.
- `platform-deploy-service` owns operation state in Directus.
- A separate platform deploy executor service should own Terraform, Cloudflare,
  shell build, workspace management, provider plugins, and other code-heavy
  deployment modules.

The current NiFi runtime bootstrap creates placeholder `InvokeHTTP` processors
for the executor contract:

- `POST /internal/operations/{operation_id}/steps/prod-deploy`
- `POST /internal/operations/{operation_id}/steps/preview-deploy`
- `POST /internal/operations/{operation_id}/steps/preview-destroy`
- `POST /internal/operations/{operation_id}/steps/prod-destroy`

Those endpoints are not implemented in this repository. The placeholder service
URL is configured by `platform_deploy_executor_url` in
`platform-deploy-flow-config`. Execution is enabled through
`execution_enabled: "true"` now that the executor service is deployed.
