# go-graphstore

Deployment documentation for the GO graphstore (Blazegraph SPARQL endpoint).

Source repository: https://github.com/geneontology/go-graphstore

## Setup environment

Start the devops docker image:

```
docker rm go-graphstore || true
docker run --name go-graphstore -it geneontology/go-devops-base:tools-jammy-0.4.4 /bin/bash
cd /tmp
```

Next, do the [credentials](https://github.com/geneontology/devops-documentation/blob/main/README.setup.md#credentials) section of the general setup document.

### Resuming an existing session

If you have already set up the docker image previously:

```bash
docker container start go-graphstore
docker exec -it go-graphstore bash -c "/bin/bash"
cd /tmp/go-graphstore/provision
```

Now you can skip to the step you need.

## Setup AWS backend

Clone the repo and navigate to the provision directory:

```
git clone https://github.com/geneontology/go-graphstore.git
cd go-graphstore/provision
```

### Terraform backend

Copy and edit the backend config:

```
cp ./production/backend.tf.sample ./aws/backend.tf
emacs ./aws/backend.tf
```

Set the following value:

| Placeholder | Value |
|-------------|-------|
| `REPLACE_ME_S3_STATE_BUCKET` | `go-workspace-graphstore` |

Initialize the backend:

```
go-deploy -init --working-directory aws -verbose
```

Verify the connection (see [Test](#test)).

_At this point, we are now set up to perform basic listing and destructive operations._ See: [Destroy previous instances](#destroy-previous-instances).

## Setup instance

### config-instance.yaml

Copy and edit the instance config:

```
cp ./production/config-instance.yaml.sample config-instance.yaml
emacs config-instance.yaml
```

#### For production

| Placeholder | Value |
|-------------|-------|
| `REPLACE_ME_INSTANCE_NAME` | `graphstore-production-YYYY-MM-DD` |
| `REPLACE_ME_DNS_RECORD_NAME` | `graphstore-production-YYYY-MM-DD.geneontology.org` |
| `REPLACE_ME_DNS_ZONE_ID` | `Z04640331A23NHVPCC784` (geneontology.org) |

#### For "internal"

| Placeholder | Value |
|-------------|-------|
| `REPLACE_ME_INSTANCE_NAME` | `graphstore-internal-YYYY-MM-DD` |
| `REPLACE_ME_DNS_RECORD_NAME` | `graphstore-internal-YYYY-MM-DD.berkeleybop.io` |
| `REPLACE_ME_DNS_ZONE_ID` | `ZNM42D2G0HNUI` (berkeleybop.io) |

### Deploy instance

Replace `YYYY-MM-DD` with today's date throughout.

#### For production

```
go-deploy --workspace production-YYYY-MM-DD --working-directory aws -verbose --conf config-instance.yaml
```

#### For "internal"

```
go-deploy --workspace internal-YYYY-MM-DD --working-directory aws -verbose --conf config-instance.yaml
```

## Setup services on instance

### config-stack.yaml

Copy and edit the stack config:

```
cp ./production/config-stack.yaml.sample ./config-stack.yaml
emacs ./config-stack.yaml
```

### Production variables

#### config-stack.yaml

| Placeholder | Value |
|-------------|-------|
| `REPLACE_ME_S3_PREFIX` | `production-YYYY-MM-DD` |
| `REPLACE_ME_S3_BUCKET` | `go-service-logs-graphstore-production` |
| `REPLACE_ME_JOURNAL_URL` | `http://current.geneontology.org/products/blazegraph/blazegraph-production.jnl.gz` |
| `REPLACE_ME_SERVER_NAME` | `rdf.geneontology.org` |
| `REPLACE_ME_SERVER_ALIAS` | `graphstore-production-YYYY-MM-DD.geneontology.org` |

Note: `config-stack.yaml` already includes SSL and S3 credential settings. The values in it override `vars.yaml` and `ssl-vars.yaml`, so you do not need to edit those files separately.

### Internal variables

#### config-stack.yaml

```
cp ./production/config-stack.yaml.sample ./config-stack.yaml
emacs ./config-stack.yaml
```

| Placeholder | Value |
|-------------|-------|
| `REPLACE_ME_S3_PREFIX` | `internal-YYYY-MM-DD` |
| `REPLACE_ME_S3_BUCKET` | `go-service-logs-graphstore-internal` |
| `REPLACE_ME_JOURNAL_URL` | `http://current.geneontology.org/products/blazegraph/blazegraph-internal.jnl.gz` |
| `REPLACE_ME_SERVER_NAME` | `rdf-internal.berkeleybop.io` |
| `REPLACE_ME_SERVER_ALIAS` | `graphstore-internal-YYYY-MM-DD.berkeleybop.io` |

### Verify configuration

Before deploying, scan the edited config files for any remaining placeholder values to ensure all variables have been properly set:

```
grep -rn 'REPLACE_ME_' config-stack.yaml config-instance.yaml ssl-vars.yaml vars.yaml aws/backend.tf
```

If any matches are found, go back and fill them in before proceeding.

## Final deployment

### Production

```
export ANSIBLE_HOST_KEY_CHECKING=False
go-deploy --workspace production-YYYY-MM-DD --working-directory aws -verbose --conf config-stack.yaml
```

### Internal

```
export ANSIBLE_HOST_KEY_CHECKING=False
go-deploy --workspace internal-YYYY-MM-DD --working-directory aws -verbose --conf config-stack.yaml
```

## Test

Verify that the backend is communicating with S3:

```
go-deploy --working-directory aws -list-workspaces -verbose
```

Verify the service is responding (replace with your DNS name):

```
wget http://graphstore-production-YYYY-MM-DD.geneontology.org/blazegraph/#query
```

## Debugging

### SSH access

From inside the devops docker image:

```
ssh -i /tmp/go-ssh ubuntu@graphstore-production-YYYY-MM-DD.geneontology.org
```

From outside the docker image:

```
ssh -i /home/sjcarbon/local/share/secrets/go/ssh-keys/go-ssh ubuntu@graphstore-production-YYYY-MM-DD.geneontology.org
```

### Docker services on the instance

All commands assume you are SSH'd into the instance. The stage directory is typically `/home/ubuntu/stage_dir`.

```
docker-compose -f /home/ubuntu/stage_dir/docker-compose.yaml ps
docker-compose -f /home/ubuntu/stage_dir/docker-compose.yaml logs -f
docker-compose -f /home/ubuntu/stage_dir/docker-compose.yaml down
docker-compose -f /home/ubuntu/stage_dir/docker-compose.yaml up -d
```

### go-deploy utility commands

The `go-deploy` script wraps Terraform and Ansible for day-to-day operations. Use these commands for routine inspection:

Preview commands without executing:

```
go-deploy --workspace production-YYYY-MM-DD --working-directory aws -verbose -dry-run --conf config-stack.yaml
```

Display the terraform state:

```
go-deploy --workspace production-YYYY-MM-DD --working-directory aws -verbose -show
```

Display the public IP address:

```
go-deploy --workspace production-YYYY-MM-DD --working-directory aws -verbose -output
```

List workspaces:

```
go-deploy --working-directory aws -list-workspaces -verbose
```

### Lower-level terraform commands

For deeper debugging, you can use Terraform directly. These commands operate on the same state as `go-deploy` but provide more detail:

List workspaces:

```
terraform -chdir=aws workspace list
```

Select and inspect a workspace:

```
terraform -chdir=aws workspace select production-YYYY-MM-DD
terraform -chdir=aws show
```

Inspect generated files:

```
cat production-YYYY-MM-DD.tfvars.json
cat production-YYYY-MM-DD-inventory.cfg
```

## Destroy previous instances

To destroy the setup associated with workspace "production-YYYY-MM-DD", use `go-deploy`:

```
go-deploy --workspace production-YYYY-MM-DD --working-directory aws -verbose -destroy
```

If finer control is needed, you can use Terraform directly to destroy and clean up the workspace:

```
terraform -chdir=aws workspace select production-YYYY-MM-DD
terraform -chdir=aws show
terraform -chdir=aws destroy
terraform -chdir=aws workspace select default
terraform -chdir=aws workspace delete production-YYYY-MM-DD
```
