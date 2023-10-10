# Setup

## Environment 

Choose _one_ of either [docker](#docker-recommended) (recommended) or [manual](#manual-good-luck) (if you insist).

### Docker (recommended)

All basic operations are designed and tested in the GO devops docker environment (i.e. all commands take place inside the docker container).
This is the recommended way to do devops.
To pin up the provided dockerized development environment:

```
docker run --name go-dev -it geneontology/go-devops-base:tools-jammy-0.4.1  /bin/bash
```

### Manual (good luck!)

#### Software

- Terraform: v1.1.4
- Ansible: 2.10.7
- aws cli
- go-deploy (multple install methods: poetry `poetry install go-deploy==0.4.2` (requires python >=3.8), can also be installed incidentally from go-fastapi repo with `poetry install`)

### Credentials

#### SSH Keys

Before starting, the credentials we generally use for deployment and access are found in the SpiderOak secret share "go-secrets".
For access and instructions, contact Seth. 

Assuming that you're using the docker image as above, and that you have the secrets at `local/share/secrets/go/ssh-keys`,
from your host machine you can copy the secrets in with.

```
docker cp local/share/secrets/go/ssh-keys/go-ssh go-dev:/tmp
docker cp local/share/secrets/go/ssh-keys/go-ssh.pub go-dev:/tmp
```

#### AWS

These credentials are used by Terraform to provision access AWS resources, such as S3 buckets and provisioning.
Copy and modify the AWS credential file to the default location `/tmp/go-aws-credentials`.

The template of the file is:

```
[default]
aws_access_key_id = REPLACE_ME_1
aws_secret_access_key = REPLACE_ME_2
```

Replace the `REPLACE_ME_1` and `REPLACE_ME_2` above in your file with your personal or project access and secret keys.
If you do not have these, or do not have ones that are sufficiently privileged, contact Seth. Assuming you have them, go
ahead and:

```bash
emacs /tmp/go-aws-credentials
```

You can test these with:

```
export AWS_SHARED_CREDENTIALS_FILE=/tmp/go-aws-credentials
aws s3 ls
```

If a list of buckets is returned, you're good.

## DNS / Cloudflare (you may not need this if just doing queries or destructive operations): 

Two DNS records are used for most services; they are typically the "production" record and the dev/testing record.

Pattern; where SERVICE is a used service CNAME/subdomain, like “api”, “rdf”, “amigo”.
go-workspace-SERVICE
SERVICE-a
SERVICE-b
one of these will be prod; the other will be either the fallback or the next; if for some reason we need both fallback and next, SERVICE-c could be used, but would indicate that we’re climbing over ourselves
SERVICE-dev-DEVNAME(-ISODATE/METADATA)

The go-deploy tool allows for creating DNS records (type A) that would be populated by the public ip addresses of the aws instance. If you don't use this option, you would need to point this record to the elastic IP of the VM. For testing purposes, you can use: `aes-test-go-fastapi.geneontology.org` or any other record that you create in Route 53.

**NOTE**: If using cloudflare, you would need to point the cloudflare dns record to the elastic IP.

## Initialization / Connection

In order to perform operations with Terraform and recover the group shared state from S3, we'll first need to initialize the
Terraform backend.

A generic backend file could look like:

```
terraform {
  backend "s3" {
    bucket  = "REPLACE_ME_TERRAFORM_S3_STATE_STORE"
    profile = "default"
    key     = "goapi/terraform.tfstate"
    region  = "us-east-1"
    encrypt = true
  }
}
```

While a lot of this is obviously generic, you'll need two things specific to your service: the `bucket` and the `key`.

`bucket` is likely the name of the S3 bucket where your service/stack resides. In general, we'll refer to this as the backend
store and it will follow the pattern of `go-workspace-SERVICE`, where SERVICE is a CNAME/subdomain of geneontology.org,
like “api”, “rdf”, “amigo”.

Often, a specific project will have a specific template to use for this file and you will initialize it inside the docker image,
inside the project repo, with a command like:
```
cp ./production/backend.tf.sample ./aws/backend.tf
```
You would then replace the REPLACE_ME_TERRAFORM_S3_STATE_STORE with the appropriate S3 bucket name:
```
emacs ./aws/backend.tf
```
Finally, let's hookup terraform to the S3 bucket state store within the ./aws directory (the `--working-directory` variable):

```
go-deploy -init --working-directory aws -verbose
```

To test, let's list the workspaces found in this state store:
```
go-deploy --working-directory aws -list-workspaces -verbose 
```

## Next steps

Now that you're setup, you can look at basic Terraform operations at README.services.

## Troubleshooting

Unsure why, but got errors when doing "real" operations until I did:

```
export AWS_REGION=us-east-1
```
