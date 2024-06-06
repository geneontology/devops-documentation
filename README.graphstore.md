# go-graphstore

## Setup environment

Using `go-workspace-graphstore` as the S3 bucket in `backend.tf`, setup as:

https://github.com/geneontology/devops-documentation/blob/main/README.setup.md

Then:

```
docker
docker run --name go-dev -it geneontology/go-devops-base:tools-jammy-0.4.2  /bin/bash
cd /tmp
```

## Setup backend and instance

Get local repo.

```
cd /tmp
git clone https://github.com/geneontology/go-graphstore.git
cd go-graphstore/provision
```

### Terraform backend

```
cp ./production/backend.tf.sample ./aws/backend.tf
emacs ./aws/backend.tf
```

- `bucket  = "REPLACE_ME_TERRAFORM_S3_STATE_STORE"` should be "go-workspace-graphstore"

Setup AWS backend with:

```
go-deploy -init --working-directory aws -verbose
```

Test with:

```
go-deploy --working-directory aws -list-workspaces -verbose
```

_At this point, we are now setup to perform basic listing and destructive operations._

### config-instance.yaml

Edit instance config.

```
cp ./production/config-instance.yaml.sample config-instance.yaml
emacs config-instance.yaml
```

#### For production

- `Name: REPLACE_ME` should be "graphstore-production-YYYY-MM-DD".
- `dns_record_name: REPLACE_ME` should be "graphstore-production-YYYY-MM-DD.geneontology.org"
- `dns_zone_id: REPLACE_ME` should be "Z04640331A23NHVPCC784" (for geneontology.org).

#### For "internal"

- `Name: REPLACE_ME` should be "graphstore-internal-YYYY-MM-DD".
- `dns_record_name: REPLACE_ME` should be "graphstore-internal-YYYY-MM-DD.berkeleybop.io"
- `dns_zone_id: REPLACE_ME` should be "ZNM42D2G0HNUI" (for berkeleybop.io).

### Deploy

Commands to deploy instance.

#### For production

```
go-deploy --workspace production-YYYY-MM-DD --working-directory aws -verbose --conf config-instance.yaml
```

#### For "internal"

```
go-deploy --workspace internal-YYYY-MM-DD --working-directory aws -verbose --conf config-instance.yaml
```

### Troubleshooting instance setup

Assume that we want to examine the "hardware" and settings around the workspace entry "production-YYYY-MM-DD", deployed to "geneontology.org".

Getting in (from outside the docker images):

```
ssh -i /home/sjcarbon/local/share/secrets/go/ssh-keys/go-ssh ubuntu@production-2024-04-24.geneontology.org
```

Workspace placement:

```
terraform -chdir=aws workspace list
```

Display the terraform state:

```
go-deploy --workspace production-YYYY-MM-DD --working-directory aws -verbose -show
```

Display the public ip address of the aws instance:

```
go-deploy --workspace production-YYYY-MM-DD --working-directory aws -verbose -output
```

The deploy command creates a terraform tfvars. These variables override the variables in `aws/main.tf`.

```
cat production-YYYY-MM-DD.tfvars.json
```

```
cat production-YYYY-MM-DD-inventory.cfg
```

## Setup services on instance

### Production variables

```
cp ./production/config-stack.yaml.sample ./config-stack.yaml
emacs ./config-stack.yaml
```

- `S3_PREFIX: REPLACE_ME` should be "production-2024-04-24".
- `S3_BUCKET: REPLACE_ME` should be "go-service-logs-graphstore-production".
- `remote_gzip_journal` should be "http://current.geneontology.org/products/blazegraph/blazegraph-production.jnl.gz"
- `GRAPHSTORE_SERVER_NAME: REPLACE_ME` should be "rdf.geneontology.org".
- `GRAPHSTORE_SERVER_ALIAS: REPLACE_ME` should be "graphstore-production-2024-04-24.geneontology.org".

Next, get the ssl certs setup:

```
emacs ./ssl-vars.yaml
```

- `USE_SSL: 0` should be "1".
- `S3_SSL_CERTS_LOCATION: REPLACE_ME` should be "s3://go-service-lockbox/geneontology.org.tar.gz".

Finally, setup logging locations:

```
emacs ./vars.yaml
```

- `S3_PATH: graphstore` should be "production-2024-04-24".
- `S3_PREFIX: REPLACE_ME` should be "production-2024-04-24".
- `S3_CRED_FILE: REPLACE_ME` should be "/tmp/go-aws-credentials".
- `S3_BUCKET: REPLACE_ME` should be "go-service-logs-graphstore-production".
- `remote_journal_gzip: http://current.geneontology.org/products/blazegraph/blazegraph-internal.jnl.gz` should be "http://current.geneontology.org/products/blazegraph/blazegraph-production.jnl.gz"
- `GRAPHSTORE_SERVER_NAME: graphstore.example.com` should be "rdf.geneontology.org".
- `S3_SSL_CERTS_LOCATION` should be "s3://go-service-lockbox/geneontology.org.tar.gz".

### Internal variables

```
cp ./production/config-stack.yaml.sample ./config-stack.yaml
emacs ./config-stack.yaml
```

- `S3_PREFIX: REPLACE_ME` should be "internal-2024-04-24".
- `S3_BUCKET: REPLACE_ME` should be "go-service-logs-graphstore-internal".
- `remote_gzip_journal` should be "http://current.geneontology.org/products/blazegraph/blazegraph-internal.jnl.gz"
- `GRAPHSTORE_SERVER_NAME: REPLACE_ME` should be "rdf-internal.berkeleybop.io".
- `GRAPHSTORE_SERVER_ALIAS: REPLACE_ME` should be "graphstore-internal-2024-04-24.berkeleybop.io".
- `S3_SSL_CERTS_LOCATION` should be "s3://go-service-lockbox/berkeleybop.io.tar.gz".

Next, get the ssl certs setup:

```
emacs ./ssl-vars.yaml
```

- `USE_SSL: 0` should be "1".
- `S3_SSL_CERTS_LOCATION: REPLACE_ME` should be "s3://go-service-lockbox/berkeleybop.io.tar.gz".

Finally, setup logging locations:

```
emacs ./vars.yaml
```

- `S3_PATH: graphstore` should be "internal-2024-04-24".
- `S3_PREFIX: REPLACE_ME` should be "internal-2024-04-24".
- `S3_CRED_FILE: REPLACE_ME` should be "/tmp/go-aws-credentials".
- `S3_BUCKET: REPLACE_ME` should be "go-service-logs-graphstore-internal".
- `remote_journal_gzip: http://current.geneontology.org/products/blazegraph/blazegraph-internal.jnl.gz` requires _no_ change.
- `GRAPHSTORE_SERVER_NAME: graphstore.example.com` should be "rdf-internal.berkeleybop.io".

## Final deployment

### Production

```
export ANSIBLE_HOST_KEY_CHECKING=False
go-deploy --workspace production-2024-04-24 --working-directory aws -verbose --conf config-stack.yaml
```

### Internal

```
export ANSIBLE_HOST_KEY_CHECKING=False
go-deploy --workspace internal-2024-04-24 --working-directory aws -verbose --conf config-stack.yaml
```

## Test

Test as needed

## Destroy previous instances

Let's assume that we're "hunting" the setup associated with "production-YYYY-MM-DD".

```
terraform -chdir=aws workspace select production-YYYY-MM-DD
terraform -chdir=aws show
terraform -chdir=aws destroy
terraform -chdir=aws workspace select default
terraform -chdir=aws workspace delete production-YYYY-MM-DD
```
