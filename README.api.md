# go-fastapi

Deployment documentation for the GO API (FastAPI).

Source repository: https://github.com/geneontology/go-fastapi

## Before you begin

Gather the following before starting. These are user- and environment-specific and cannot be derived from the repo:

| Item | Description | Example |
|------|-------------|---------|
| AWS credentials file | Path on your host machine | `/home/you/secrets/go-aws-credentials` |
| SSH private key | Path on your host machine | `/home/you/secrets/go-ssh` |
| SSH public key | Path on your host machine | `/home/you/secrets/go-ssh.pub` |
| Date stamp | Today's date in YYYY-MM-DD format | `2026-03-12` |

For credential details (format, where to get them), see the [credentials](https://github.com/geneontology/devops-documentation/blob/main/README.setup.md#credentials) section of the general setup document.

Once inside the devops container, these files must be at:
- `/tmp/go-aws-credentials`
- `/tmp/go-ssh`
- `/tmp/go-ssh.pub`

## Setup environment

Start the devops docker image:

```
docker rm go-fastapi || true
docker run --name go-fastapi -it geneontology/go-devops-base:tools-jammy-0.4.4  /bin/bash
cd /tmp
```

Next, do the [credentials](https://github.com/geneontology/devops-documentation/blob/main/README.setup.md#credentials) section of the general setup document.

### Resuming an existing session

If you have already set up the docker image previously:

```bash
docker container start go-fastapi
docker exec -it go-fastapi bash -c "/bin/bash"
cd /tmp/go-fastapi/provision
```

Now you can skip to the step you need.

## Releasing (before deploying new code)

A GitHub release must be created before deploying new code. The release tag triggers CI to build and push a Docker image to DockerHub.

```
gh release create vX.Y.Z --target main --generate-notes --repo geneontology/go-fastapi
```

This triggers the Docker build workflow, producing `geneontology/go-fastapi:X.Y.Z` on DockerHub. Use `X.Y.Z` (no leading "v") as the `REPLACE_ME_FASTAPI_TAG` value when deploying.

See previous releases at: https://github.com/geneontology/go-fastapi/releases

## Setup AWS backend

Clone the repo and navigate to the provision directory:

```
git clone https://github.com/geneontology/go-fastapi.git
cd go-fastapi/provision
```

### Terraform backend

Copy and edit the backend config:

```
cp ./production/backend.tf.sample ./aws/backend.tf
emacs ./aws/backend.tf
```

| Placeholder | Value |
|-------------|-------|
| `REPLACE_ME_GOAPI_S3_STATE_STORE` | `go-workspace-api` |

Initialize the backend:

```
go-deploy -init --working-directory aws -verbose
```

Verify the connection (see [Test](#test)).

_At this point, we are now set up to perform basic listing and destructive operations._ See: [Destroy previous instances](#destroy-previous-instances).

## Setup instance

### Choose your variable

Use the following namespace pattern `go-api-production-YYYY-MM-DD`; e.g.: `go-api-production-2024-10-23`. We'll use `go-api-production-YYYY-MM-DD` as the stable shortcut for this.

### config-instance.yaml

Copy and edit the instance config:

```
cp ./production/config-instance.yaml.sample config-instance.yaml
emacs config-instance.yaml
```

| Placeholder | Value |
|-------------|-------|
| `REPLACE_ME_INSTANCE_NAME` | `go-api-production-YYYY-MM-DD` |
| `REPLACE_ME_DNS_RECORD` | `go-api-production-YYYY-MM-DD.geneontology.org` |
| `REPLACE_ME_DNS_ZONE_ID` | `Z04640331A23NHVPCC784` (geneontology.org) |

### Deploy instance

Replace `YYYY-MM-DD` below appropriately, as outlined above.

To test:

```
go-deploy --workspace go-api-production-YYYY-MM-DD --working-directory aws -verbose --dry-run --conf config-instance.yaml
```

Deploy:

```
go-deploy --workspace go-api-production-YYYY-MM-DD --working-directory aws -verbose --conf config-instance.yaml
```

## Setup services on instance

### config-stack.yaml

Copy and edit the stack config:

```bash
cp ./production/config-stack.yaml.sample ./config-stack.yaml
emacs ./config-stack.yaml
```

| Placeholder | Value |
|-------------|-------|
| `REPLACE_ME_APACHE_LOG_BUCKET` | `go-service-logs-api` |
| `REPLACE_ME_SSL_CERTS_LOCATION` | `s3://go-service-lockbox/geneontology.org.tar.gz` |
| `REPLACE_ME_HOST_ALIAS` | `go-api-production-YYYY-MM-DD.geneontology.org` |
| `REPLACE_ME_FASTAPI_TAG` | Dockerhub tagged version (GitHub release sans leading "v"); see [releases](https://github.com/geneontology/go-fastapi/releases) |

Note: `config-stack.yaml` already includes SSL and S3 credential settings. The values in it override `vars.yaml` and `ssl-vars.yaml`, so you do not need to edit those files separately.

### Verify configuration

Before deploying, scan the edited config files for any remaining placeholder values:

```
grep -rn 'REPLACE_ME_' config-stack.yaml config-instance.yaml aws/backend.tf
```

If any matches are found, go back and fill them in before proceeding.

### Deploy stack

```bash
export ANSIBLE_HOST_KEY_CHECKING=False
go-deploy --workspace go-api-production-YYYY-MM-DD --working-directory aws -verbose --conf config-stack.yaml
```

If you're satisfied (have tested) the instance at https://go-api-production-YYYY-MM-DD.geneontology.org, you can finalize by aiming the Cloudflare for api.geneontology.org at this newly created instance. You will need the IP address (listed as `public_ip`) from:

```bash
go-deploy --workspace go-api-production-YYYY-MM-DD --working-directory aws -verbose -show
```

Once settling is completed, proceed to remove the previous instance(s) below.

## Test

Verify that the backend is communicating with S3:

```
go-deploy --working-directory aws -list-workspaces -verbose
```

Verify the service is responding (replace with your DNS name):

```
curl -s https://go-api-production-YYYY-MM-DD.geneontology.org/docs | head -20
```

## Debugging

### SSH access

From inside the devops docker image:

```
ssh -i /tmp/go-ssh ubuntu@go-api-production-YYYY-MM-DD.geneontology.org
```

### Docker services on the instance

All commands assume you are SSH'd into the instance. The stage directory is `/home/ubuntu/stage_dir`.

```
docker-compose -f /home/ubuntu/stage_dir/docker-compose.yaml ps
docker-compose -f /home/ubuntu/stage_dir/docker-compose.yaml logs -f
docker-compose -f /home/ubuntu/stage_dir/docker-compose.yaml down
docker-compose -f /home/ubuntu/stage_dir/docker-compose.yaml up -d
```

### go-deploy utility commands

Preview commands without executing:

```
go-deploy --workspace go-api-production-YYYY-MM-DD --working-directory aws -verbose -dry-run --conf config-stack.yaml
```

Display the terraform state:

```
go-deploy --workspace go-api-production-YYYY-MM-DD --working-directory aws -verbose -show
```

Display the public IP address:

```
go-deploy --workspace go-api-production-YYYY-MM-DD --working-directory aws -verbose -output
```

List workspaces:

```
go-deploy --working-directory aws -list-workspaces -verbose
```

### Lower-level terraform commands

For deeper debugging, you can use Terraform directly:

```
terraform -chdir=aws workspace list
terraform -chdir=aws workspace select go-api-production-YYYY-MM-DD
terraform -chdir=aws show
```

Inspect generated files:

```
cat go-api-production-YYYY-MM-DD.tfvars.json
cat go-api-production-YYYY-MM-DD-inventory.cfg
```

## Destroy previous instances

To destroy the setup associated with "go-api-production-YYYY-MM-DD":

```
terraform -chdir=aws workspace select go-api-production-YYYY-MM-DD
terraform -chdir=aws show
terraform -chdir=aws destroy
terraform -chdir=aws workspace select default
terraform -chdir=aws workspace delete go-api-production-YYYY-MM-DD
```
