# Sync To GCP

This repo contains source code for a commandline utility that either monitors local or nfs filesystems for changes by interval based polling, or filesystem events depending on system architecture. Currently, this tool supports syncing a collection of files or contents of directories to a Google Cloud Storage Bucket, or an individual file's contents to a Google Secret Manager Secret as a new version. This tool will only sync changes when differences are detected between the copy in GCP and the local copy.

## Prerequisites

### Auth

Users and automated systems needing to execute this application will need to be authenticated to GCP either by setting the application-default credentials, or by exporting a service account json key to `GOOGLE_APPLICATION_CREDENTIALS`. See below for examples.

#### Application Default Credentials

```bash
# login with application-default credentials
gcloud auth application-default login
```

#### Service Account JSON Key

```bash
# authenticate using a service account json key
export GOOGLE_APPLICATION_CREDENTIALS=./key/to/service/account.json
```

## Usage

Download a binary from the [latest release](https://github.com/ammilam/file-sync-to-gcp/releases/tag/latest). Depending on the intended destination of the local file and/or its contents, refer to [Google Cloud Storage](#google-cloud-storage) or [Google Secret Manager](#google-secret-manager-secret).

### Google Cloud Storage

When syncing to Google Cloud Storage, the following flags are supported:

- path => (required) path to the local file or directory to sync to gcp, multiple can be specified
- bucket => (required) google storage bucket to upload the file(s) to
- bucket_path => (optional) path on the google storage bucket to upload the file(s) to, defaults to the root of the bucket
- interval => (optional) sets the interval in seconds to poll the directory for changes, filesystem events are used by default
- type => (optional) accepts either cloud-storage or secret-manager
- method => (optional) sets if the file is moved or copied to gcs, accepts `move` and `copy`, defaults to copy

```bash
##########################
## --type=cloud-storage ##
##########################
# copy to the root of the bucket
./file-sync-to-gcp  --path=./path/to/local/dir --bucket=gcs-bucket-name

# copy to a specific directory on a bucket
./file-sync-to-gcp  --path=./path/to/local/dir --bucket=gcs-bucket-name --bucket_path=path/on/bucket

# specifying multiple paths
./file-sync-to-gcp --path=./path/to/local/file.txt --path=./path/to/another/file.txt --bucket=gcs-bucket-name --interval=900

# moving a file
./file-sync-to-gcp --path=./path/to/local/file.txt --bucket=gcs-bucket-name --interval=900 --method=move

```

### Google Secret Manager Secret

When syncing to a Google Secret Manager Secret, the following flags are supported:

- path => (required) path to a single local file, directories are not supported
- secret => (required for secret-manager) a secret manager secret name
- project => (required for secret-manager) the google project id containing the secret manager secret
- interval => (optional) sets the interval in seconds to poll the directory for changes
- type => (optional) accepts either cloud-storage or secret-manager

```bash
###########################
## --type=secret-manager ##
###########################
# only accepts files, not folders
./file-sync-to-gcp --path=./path/to/cert.pem --secret=private-key --project=a-gcp-project-1234
```

## Running In Docker

```bash
# login via application default credentials (or get a service account.json key)
gcloud auth application-default login

# copy json key file to the current working directory
cp /Users/a-user/.config/gcloud/application_default_credentials.json ./account.json

# run the container
docker run \
  --volume $PWD:/mount \
  --detach \
  --env GOOGLE_APPLICATION_CREDENTIALS=/mount/account.json \
  ghcr.io/ammilam/file-sync-to-gcp:latest \
  "./file-sync-to-gcp --path=./mount/path/to/file.txt --bucket=a-gcs-bucket --interval=300"
```

## Helm

In order to get started deploying this solution via helm, you will need to provide credentials for the pod to work with. The helm chart, which can be pulled from the [ammilam helm chart repo](https://ammilam.github.io/), supports ingesting GCP credentials from a vault agent side car container, or by having a kubernetes secret with a static account.json key present in the secret data for reference.

>values.yaml

```yaml
# command to run at startup
command: |-
  ./file-sync-to-gcp --path=./data/test.txt --bucket=test-bucket-1234
  
# mount a nfs to the pod
nfs:
  server: "a-host.company.corp"
  server_path: "/path/on/server"
  container_mount_path: "/data"
  
# used to mount a gcp service account json key to the pod
gcp_credentials:
  # where the mount the gcp service account json key in the pod
  mount_path: /credentials
  key_file_name: account.json

  # used for static service account json file mounted as a k8s secret, this is not necessary when using vault
  k8s_secret_service_account_json: "gcp-service-account-k8s-secret"

  # if vault is available with approle auth method and roleset integration to gcp
  # will dynamically mount and rotate service account json file to the pod
  vault:
    enable_vault_agent: false # set to true if using vault agent
    secret: "a-k8s-secret" # should have role_id="a-vault-role-id" secret_id="a-vault-secret-id" in the secret data
    vault_addr: https://vault.company.com
    mount_path: auth/approle
    vault_namespace: ""    # if using a vault namespace
    secret_endpoint: /gcp/key/a-gcp-roleset
```

Once values.yaml has been configured, install the chart.

```bash
# add the ammilam helm repo
helm repo add ammilam https://ammilam.github.io/helm-charts/

# install the helm chart using the values.yaml file
helm upgrade --install file-sync-to-gcp ammilam/file-sync-to-gcp -f values.yaml
```

- Helm Chart Source: https://github.com/ammilam/helm-charts/file-sync-to-gcp
- Helm Chart Repo: https://ammilam.github.io/helm-charts/
