# mlflow

![mlflow](https://raw.githubusercontent.com/mlflow/mlflow/master/docs/source/_static/MLflow-logo-final-black.png)

![Version: 0.1.0](https://img.shields.io/badge/Version-0.1.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 2.10.0](https://img.shields.io/badge/AppVersion-2.10.0-informational?style=flat-square)

A Helm chart for MLflow - Open source platform for the machine learning lifecycle.

**Homepage:** <https://github.com/mlflow/mlflow/tree/master/charts/mlflow>

## Prerequisites

- Kubernetes >= 1.19
- Helm >= 3.2.0

## How To Use

### Installing the Chart

To install the chart with release name `my-release`:

```bash
helm install my-release mlflow/mlflow
```

### Upgrading the Chart

To upgrade the chart with release name `my-release`:

```bash
helm upgrade my-release mlflow/mlflow
```

### Uninstalling the chart

To uninstall the chart with release name `my-release`:

```bash
helm uninstall my-release
```

## Configurations

### Backend Store

For more information, please visit [Supported Backend Store Types](https://mlflow.org/docs/latest/tracking/backend-stores.html#supported-store-types).

#### Use Database as Backend Store

Database can be configured as backend store in the following two ways:

1. Create a secret to store database connection url and refer it in `valus.yaml`.
2. Configure backend store url in `values.yaml` and a new secret will be created to store it.

First, create a secret to store database connection url. The following command create a secret with name `mlflow-database-backend-store-secret`:

```bash
kubectl create secret generic mlflow-database-backend-store-secret \
    --from-literal=BACKEND_STORE_URL=<dialect>+<driver>://<username>:<password>@<host>:<port>/<database>
```

For more details, please visit [SQLAlchemy Database URL](https://docs.sqlalchemy.org/en/latest/core/engines.html#database-urls).

Then configure `values.yaml` as following:

```yaml
backendStore:
  existingSecret: mlflow-mysql-backend-store-secret
```

Otherwise, if no existing secret is specified and `backendStore.createSecret.backendStoreUri` is given, then creates a secret to store it:

```yaml
backendStore:
  createSecret:
    backendStoreUri: <dialect>+<driver>://<username>:<password>@<host>:<port>/<database>
```

### Artifact Store

Supported storage types for the artifact store are:

- Amazon S3 and S3-compatible storage e.g. `s3://<bucket>/<path>`
- Azure Blob Storage e.g. `wasbs://<container>@<storage-account>.blob.core.windows.net/<path>`
- Google Cloud Storage e.g. `gs://<bucket>/<path>`
- Alibaba Cloud OSS e.g. `oss://<bucket>/<path>`
- FTP server e.g. `ftp://user:pass@host/path/to/directory`
- SFTP Server e.g. `sftp://user@host/path/to/directory`
- NFS e.g. `/mnt/nfs`
- HDFS e.g. `hdfs://<host>:<port>/<path>`

#### Configure Default Artifact Root

The following configuration enables artifact store and set default artifact root as `./mlruns`:

```yaml
artifactStore:
  enabled: true
  defaultArtifactRoot: ./mlruns
```

#### Use AWS S3 for Artifact Store

There are two different ways to connect to S3:

1. Access S3 with AWS IAM role ARN by adding annotations to the service account.
2. Access S3 with AWS IAM user's `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.

##### Use AWS IAM Role ARN

Associate an AWS IAM role to the service account by adding annotations as follows:

```yaml
serviceAccount:
  create: true
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::account-id:role/<YOUR_IAM_ROLE_ARN>"
```

For detailed information, please visit [Configuring a Kubernetes service account to assume an IAM role - Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/associate-service-account-role.html).

##### Use AWS Access Key

Create a secret to store S3 access credentials:

```bash
kubectl create secret generic mlflow-s3-artifact-store-secret \
    --from-literal=AWS_ACCESS_KEY_ID=<YOUR_AWS_ACCESS_KEY_ID> \
    --from-literal=AWS_SECRET_ACCESS_KEY=<YOUR_AWS_SECRET_ACCESS_KEY>
```

Then you can configure `values.yaml` as following:

```yaml
artifactStore:
  enabled: true
  s3:
    enabled: true
    existingSecret: mlflow-s3-artifact-store-secret
```

Otherwise, if no existing credentials secret is specified, it will create a secret for you used to connect to S3:

```yaml
artifactStore:
  enabled: true
  defaultArtifactStore: s3://<bucket>/<path>
  s3:
    enabled: true
    createSecret:
      accessKeyId: <YOUR_AWS_ACCESS_KEY_ID>
      secretAccessKey: <YOUR_AWS_SECRET_ACCESS_KEY>
```

##### Extra Configurations

To add S3 file upload extra arguments, you need to set `MLFLOW_S3_UPLOAD_EXTRA_ARGS` to a JSON object of key/value pairs. For example, if you want to upload to a KMS Encrypted bucket using the KMS Key 1234:

```yaml
extraEnv:
- name: MLFLOW_S3_UPLOAD_EXTRA_ARGS
  value: |
    {"ServerSideEncryption": "aws:kms", "SSEKMSKeyId": "1234"}
```

To store artifacts in a custom S3 endpoint, you need to set `MLFLOW_S3_ENDPOINT_URL` environment variable as follows:

```yaml
extraEnv:
- name: MLFLOW_S3_ENDPOINT
  value: <YOUR_S3_ENDPOINT_URL>
```

If you want to disable TLS authentication, you can set `MLFLOW_S3_IGNORE_TLS` variable to `true`:

```yaml
extraEnv:
- name: MLFLOW_S3_IGNORE_TLS
  value: true
```

Additionally, if MinIO server is configured with non-default region, you should set `AWS_DEFAULT_REGION` variable:

```yaml
extraEnv:
- name: AWS_DEFAULT_REGION
  value: <YOUR_REGION>
```

#### Use Azure Blob Storage for Artifact Store

Create a secret to store Azure access credentials:

```yaml
kubectl create secret generic mlflow-azure-artifact-store-secret \
    --from-literal=AZURE_STORAGE_CONNECTION_STRING=<AZURE_STORAGE_CONNECTION_STRING> \
    --from-literal=AZURE_STORAGE_ACCESS_KEY=<YOUR_AZURE_STORAGE_ACCESS_KEY>
```

Then configure Azure artifact store to use the secret just created:

```yaml
artifactStore:
  enabled: true
  azure:
    enabled: true
    existingSecret: mlflow-azure-artifact-store-secret
```

#### Use Google Cloud Storage for Artifact Store

Create a secret to store GCP access credentials:

```yaml
kubectl create secret generic mlflow-gcp-artifact-store-secret \
    --from-literal=keyfile.json=<YOUR_KEYFILE_CONTENT>
```

```yaml
artifactStore:
  enabled: true
  defaultArtifactRoot: gs://<bucket>/<path>
  gcp:
    enabled: true
    existingSecret: mlflow-gcp-artifact-store-secret
```

You may set some MLflow environment variables to troubleshoot GCS read-timeouts by setting the following variables:

```yaml
extraEnv:
# Sets the standard timeout for transfer operations in seconds (Default: 60 for GCS). Use -1 for indefinite timeout.
- name: MLFLOW_ARTIFACT_UPLOAD_DOWNLOAD_TIMEOUT
  value: 60
# Sets the standard upload chunk size for bigger files in bytes (Default: 104857600 ≙ 100MiB), must be multiple of 256 KB.
- name: MLFLOW_GCS_UPLOAD_CHUNK_SIZE
  value: 104857600
# Sets the standard download chunk size for bigger files in bytes (Default: 104857600 ≙ 100MiB), must be multiple of 256 KB.
- name: MLFLOW_GCS_DOWNLOAD_CHUNK_SIZE
  value: 104857600
```

#### Use Alibaba Cloud OSS for Artifact Store

There are two different ways to access OSS:

1. Associates an Alibaba Cloud RAM role to the service account **(Recommended)**.
2. Use Alibaba Cloud RAM user's AccessKey ID and AccessKey Secret.

##### Use Alibaba Cloud RAM Role

Associate an Alibaba Cloud RAM role to the service account by adding annotations as follows:

```yaml
serviceAccount:
  create: true
  annotations:
    pod-identity.alibabacloud.com/role-name: <YOUR_ALIBABA_CLOUD_RAM_ROLE_NAME>
```

For more information, please visit [Use RRSA to authorize different pods to access different cloud services - Container Service for Kubernetes - Alibaba Cloud Documentation Center](https://www.alibabacloud.com/help/ack/ack-managed-and-ack-dedicated/user-guide/use-rrsa-to-authorize-pods-to-access-different-cloud-services).

##### Use Alibaba Cloud AccessKey

Create a secret to store Alibaba Cloud AccessKey ID and AccessKey Secret as follows:

```bash
kubectl create secret generic mlflow-oss-artifact-store-secret \
    --from-literal=MLFLOW_OSS_KEY_ID=<ALIBABA_CLOUD_ACCESS_KEY_ID> \
    --from-literal=MLFLOW_OSS_KEY_SECRET=<ALIBABA_CLOUD_ACCESS_KEY_SECRET>
```

Then configure OSS artifact store to use the secret just created:

```yaml
artifactStore:
  enabled: true
  defaultArtifactRoot: oss://<bucket>/<path>
  oss:
    enabled: true
    existingSecret: mlflow-oss-artifact-store-secret
```

Otherwise, if no existing secret is specified, you can fill your access credentials as follows and a new secret will be created to store it:

```yaml
artifactStore:
  enabled: true
  defaultArtifactRoot: oss://<bucket>/<path>
  oss:
    enabled: true
    createSecret:
      accessKeyId: <YOUR_ALIBABA_CLOUD_ACCESS_KEY_ID>
      accessKeySecret: <YOUR_ALIBABA_CLOUD_ACCESS_KEY_SECRET>
```

### Authentication

### Use BasicAuth

MLflow supports basic HTTP authentication to enable access control over experiments and registered models.

Suppose you have `basic_auth.ini` file as follows:

```ini
[mlflow]
default_permission = READ
database_uri = sqlite:///basic_auth.db
admin_username = admin
admin_password = password
authorization_function = mlflow.server.auth:authenticate_request_basic_auth
```

Create a secret to store basic auth configurations from configuration file:

```bash
kubectl create secret generic mlflow-basic-auth-secret --from-file=basic_auth.ini
```

Then enable BasicAuth and use the secret just created:

```yaml
trackingServer:
  enabled: true
  basicAuth:
    enabled: true
    existingSecret: mlflow-basic-auth-secret
```

Otherwise, you can directly configure basic authentication as follows:

```yaml
trackingServer:
  enabled: true
  basicAuth:
    enabled: true
    createSecret:
      defaultPermission: READ
      databaseUri: sqlite:///basic_auth.db
      adminUsername: admin
      adminPassword: password
      authorizationFunction: mlflow.server.auth:authenticate_request_basic_auth
```

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| affinity | object | `{}` | Pod affinity |
| artifactStore.artifactsDestination | string | `""` | Specifies the base artifact location from which to resolve artifact upload/download/list requests (e.g. `s3://my-bucket`) |
| artifactStore.azure.createSecret | object | `{"azureStorageAccessKey":"","azureStorageConnectionString":""}` | If Azure is enabled as artifact store backend and no existing secret is specified, create the secret used to connect to Azure |
| artifactStore.azure.enabled | bool | `false` | Specifies whether to enable Azure Blob Storage as artifact store backend |
| artifactStore.azure.existingSecret | string | `""` | Name of an existing secret containing the key `AZURE_STORAGE_CONNECTION_STRING` or `AZURE_STORAGE_ACCESS_KEY` to store credentials to access artifact storage on AZURE |
| artifactStore.defaultArtifactRoot | string | `""` | Specifies a default artifact location for logging, default as local `./mlruns` directory |
| artifactStore.enabled | bool | `true` | Specifies whether to enable serving of artifact uploads, downloads, and list requests by routing these requests to the artifact storage service |
| artifactStore.gcp.createSecret | object | `{"keyFile":""}` | If GCP is enabled as artifact storage and no existing secret is specified, create the secret used to connect to GCP |
| artifactStore.gcp.createSecret.keyFile | string | `""` | Content of key file |
| artifactStore.gcp.enabled | bool | `false` | Specifies whether to enable Google Cloud Storage as artifact store backend |
| artifactStore.gcp.existingSecret | string | `""` | Name of an existing secret containing the key `keyfile.json` used to store credentials to access GCP |
| artifactStore.oss.createSecret | object | `{"accessKeyId":"","accessKeySecret":""}` | If OSS is enabled as artifact store backend and no existing secret is specified, create the secret used to store OSS access credentials |
| artifactStore.oss.createSecret.accessKeyId | string | `""` | Alibaba Cloud access key ID |
| artifactStore.oss.createSecret.accessKeySecret | string | `""` | Alibaba Cloud access key secret |
| artifactStore.oss.enabled | bool | `false` | Specifies whether to enable Alibaba Cloud Object Store Service(OSS) as artifact store backend |
| artifactStore.oss.endpoint | string | `""` | Endpoint of OSS e.g. oss-cn-beijing-internal.aliyuncs.com |
| artifactStore.oss.existingSecret | string | `""` | Name of an existing secret containing the key `MLFLOW_OSS_KEY_ID` and `MLFLOW_OSS_KEY_SECRET` to store credentials to access OSS |
| artifactStore.s3.createCaSecret | object | `{"caBundle":""}` | If S3 is enabled as artifact store backend and no existing CA secret is specified, create the secret used to secure connection to S3 / Minio |
| artifactStore.s3.createCaSecret.caBundle | string | `""` | Content of CA bundle |
| artifactStore.s3.createSecret | object | `{"accessKeyId":"","secretAccessKey":""}` | If S3 is enabled as artifact storage backend and no existing secret is specified, create the secret used to connect to S3 / Minio |
| artifactStore.s3.createSecret.accessKeyId | string | `""` | AWS access key ID |
| artifactStore.s3.createSecret.secretAccessKey | string | `""` | AWS secret access key |
| artifactStore.s3.enabled | bool | `false` | Specifies whether to enable AWS S3 as artifact store backend |
| artifactStore.s3.existingCaSecret | string | `""` | Name of an existing secret containing the key `ca-bundle.crt` used to store the CA certificate for TLS connections |
| artifactStore.s3.existingSecret | string | `""` | Name of an existing secret containing the keys `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` to access artifact storage on AWS S3 or MINIO |
| backendStore.createSecret | object | `{"backendStoreUri":""}` | If no existing secret is specified, creates a secret to store backend store uri |
| backendStore.createSecret.backendStoreUri | string | `""` | Backend store uri |
| backendStore.existingSecret | string | `""` | Name of an existing secret which contains key `BACKEND_STORE_URI` |
| extraContainers | list | `[]` | Extra containers belonging to the mlflow pod. |
| extraEnv | list | `[]` | Extra environment variables in mlflow container |
| extraEnvFrom | list | `[]` | Extra environment variable sources in mlflow container |
| extraInitContainers | list | `[]` | Extra initialization containers belonging to the mlflow pod. |
| extraVolumeMounts | list | `[]` | Extra volume mounts to mount into the mlflow container's file system |
| extraVolumes | list | `[]` | Extra volumes that can be mounted by containers belonging to the mlflow pod |
| fullnameOverride | string | `""` | String to override the default generated fullname |
| image.pullPolicy | string | `"IfNotPresent"` | Image pull policy |
| image.pullSecrets | list | `[]` | Image pull secrets for private docker registry |
| image.registry | string | `"ghcr.io"` | Docker image registry |
| image.repository | string | `"mlflow/mlflow"` | Docker image repository |
| image.tag | string | `""` | Docker image tag, default is `v${appVersion}` |
| ingress.annotations | object | `{}` | Annotations to add to the ingress |
| ingress.className | string | `""` | Ingress class name |
| ingress.enabled | bool | `false` | Specifies whether a ingress should be created |
| ingress.hosts | list | `[{"host":"chart-example.local","paths":[{"path":"/","pathType":"ImplementationSpecific"}]}]` | Host rules to configure the ingress |
| ingress.tls | list | `[]` | TLS configuration |
| nameOverride | string | `""` | String to override the default generated name |
| nodeSelector | object | `{}` | Pod node selector |
| podAnnotations | object | `{}` | Pod annotations |
| podSecurityContext | object | `{}` | Pod security context |
| replicaCount | int | `1` | Number of mlflow server replicas to deploy |
| resources | object | `{}` | Pod resources |
| securityContext | object | `{}` | Container security context |
| service.annotations | object | `{}` | Annotations to add to the service |
| service.name | string | `"http"` | Service port name |
| service.port | int | `5000` | Service port number |
| service.type | string | `"ClusterIP"` | Specifies which type of service should be created |
| serviceAccount.annotations | object | `{}` | Annotations to add to the service account |
| serviceAccount.create | bool | `true` | Specifies whether a service account should be created |
| serviceAccount.name | string | `""` | Name of the service account to use. If not set and create is true, a name is generated using the fullname template |
| tolerations | list | `[]` | Pod tolerations |
| trackingServer.basicAuth.createSecret.adminPassword | string | `"password"` | Default admin password if the admin is not already created |
| trackingServer.basicAuth.createSecret.adminUsername | string | `"admin"` | Default admin username if the admin is not already created |
| trackingServer.basicAuth.createSecret.authorizationFunction | string | `"mlflow.server.auth:authenticate_request_basic_auth"` | Function to authenticate requests |
| trackingServer.basicAuth.createSecret.databaseUri | string | `"sqlite:///basic_auth.db"` | Database location to store permissions and user data |
| trackingServer.basicAuth.createSecret.defaultPermission | string | `"READ"` | Default permission on all resources |
| trackingServer.basicAuth.enabled | bool | `false` | Specifies whether to enable basic authentication |
| trackingServer.basicAuth.existingSecret | string | `""` | Name of an existing secret which contains key `default_permission`, `database_uri`, `admin_username`, `admin_password` and `authorization_function` |
| trackingServer.enabled | bool | `true` | Specifies whether to enable tracking server |
| trackingServer.extraArgs | list | `[]` | Extra arguments passed to the `mlflow server` command |
| trackingServer.host | string | `"0.0.0.0"` | Network address to listen on |
| trackingServer.port | int | `5000` | Port to expose the tracking server |
| trackingServer.workers | int | `1` | Number of gunicorn worker processes to handle requests |

## Source Code

* <https://github.com/mlflow/mlflow/tree/master/charts/mlflow>

## Maintainers

| Name | Email | Url |
| ---- | ------ | --- |
| Yi Chen | <github@chenyicn.net> |  |
