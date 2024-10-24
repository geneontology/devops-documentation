# go-fastapi

## Setup environment

First, setup the devops docker image:

```
docker rm go-dev || true
docker run --name go-dev -it geneontology/go-devops-base:tools-jammy-0.4.4  /bin/bash
cd /tmp
```

Next, do the https://github.com/geneontology/devops-documentation/blob/main/README.setup.md#credentials section of the general setup document.

## Setup AWS backend

Get local repo.

```
git clone https://github.com/geneontology/go-fastapi.git
cd go-fastapi/provision
```

### Terraform backend

```
cp ./production/backend.tf.sample ./aws/backend.tf
emacs ./aws/backend.tf
```

- `bucket  = "REPLACE_ME_GOAPI_S3_STATE_STORE"` should be "go-workspace-api"

Setup AWS backend with:

```
go-deploy -init --working-directory aws -verbose
```

Test with:

See https://github.com/geneontology/devops-documentation/blob/main/README.api.md#test .

_At this point, we are now setup to perform basic listing and destructive operations._ See: https://github.com/geneontology/devops-documentation/blob/main/README.api.md#destroy-previous-instances .

## Setup instance

### Choose your variable.

Use the following namespace pattern `go-api-production-YYYY-MM-DD`; e.g.: `go-api-production-2024-10-23`. We'll use `go-api-production-YYYY-MM-DD` as the stable shortcut for this.

### config-instance.yaml

Edit instance config.

```
cp ./production/config-instance.yaml.sample config-instance.yaml
emacs config-instance.yaml
```

- `Name: REPLACE_ME` should be "go-api-production-YYYY-MM-DD".
- `dns_record_name: REPLACE_ME` should be "go-api-production-YYYY-MM-DD.geneontology.org"
- `dns_zone_id: REPLACE_ME` should be "Z04640331A23NHVPCC784" (for geneontology.org).

### Deploy

Replace `YYYY-MM-DD` below appropriately, as outlined above.

To test:

```
go-deploy --workspace go-api-production-YYYY-MM-DD --working-directory aws -verbose --dry-run --conf config-instance.yaml
```

Finally, deploy:

```
go-deploy --workspace go-api-production-YYYY-MM-DD --working-directory aws -verbose --conf config-instance.yaml
```

### Troubleshooting instance setup

Assume that we want to examine the "hardware" and settings around the workspace entry "go-api-production-YYYY-MM-DD", deployed to "geneontology.org".

Getting in (from outside the docker images):

For example:

```
ssh -i /home/sjcarbon/local/share/secrets/go/ssh-keys/go-ssh ubuntu@go-api-production-2024-04-24.geneontology.org
```

Workspace placement:

```
terraform -chdir=aws workspace list
```

Display the terraform state:

```
go-deploy --workspace go-api-production-YYYY-MM-DD --working-directory aws -verbose -show
```

Display the public ip address of the aws instance:

```
go-deploy --workspace go-api-production-YYYY-MM-DD --working-directory aws -verbose -output
```

The deploy command creates a terraform tfvars. These variables override the variables in `aws/main.tf`.

```
cat go-api-production-YYYY-MM-DD.tfvars.json
```

```
cat go-api-production-YYYY-MM-DD-inventory.cfg
```

## Setup services on instance

### Production variables

```bash
cp ./production/config-stack.yaml.sample ./config-stack.yaml
emacs ./config-stack.yaml
```

- `S3_BUCKET: REPLACE_ME_APACHE_LOG_BUCKET` should be "go-service-logs-api"
- `S3_SSL_CERTS_LOCATION: s3://REPLACE_ME/geneontology.org.tar.gz` should be "s3://go-service-lockbox/geneontology.org.tar.gz"
* `fastapi_host_alias: REPLACE_ME` should be "go-api-production-YYYY-MM-DD.geneontology.org"
* `fastapi_tag: 0.2.0`: should be the Dockerhub _tagged_ version of the API that you wish to use--this how we deploy within the image; this is conincidentally the GitHub version of the API _sans the "v"; see at https://github.com/geneontology/go-fastapi/releases .

### Deploy

```bash
go-deploy --workspace go-api-production-YYYY-MM-DD --working-directory aws -verbose --conf config-stack.yaml
```

If you're satisfied (have tested) the instance at https://go-api-production-YYYY-MM-DD.geneontology.org, you can finalize by aiming the Cloudflare for api.geneontology.org at this newly created instance. You will need the IP address (list as `public_ip`) from:

```bash
go-deploy --workspace go-api-production-2024-10-23  --working-directory aws -verbose -show
```

Once settling is completed, proceed to remove the previous instance(s)
below.

## Test

This tests that the backend is communicating with S3

```
go-deploy --working-directory aws -list-workspaces -verbose
```

## Destroy previous instances

Let's assume that we're "hunting" the setup associated with
"go-api-production-YYYY-MM-DD". The process outlined here is to
destroy the contents of the workspace, then to destroy the workspace
itself.

```
terraform -chdir=aws workspace select go-api-production-YYYY-MM-DD
terraform -chdir=aws show
terraform -chdir=aws destroy
terraform -chdir=aws workspace select default
terraform -chdir=aws workspace delete go-api-production-YYYY-MM-DD
```
