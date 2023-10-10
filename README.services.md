# Operations

## Destroying Instances and Deleting Workspaces

```bash
# Destroy Using go-deploy.
# Make sure you point to the correct workspace before destroying the stack by using the -show command or the -output command
go-deploy --workspace REPLACE_ME_WITH_TERRAFORM_BACKEND --working-directory aws -verbose -destroy
```

```bash
# Destroy Manually
# Make sure you point to the correct workspace before destroying the stack.

terraform -chdir=aws workspace list
terraform -chdir=aws workspace show # shows the name of the current workspace
terraform -chdir=aws show           # shows the state you are about to destroy
terraform -chdir=aws destroy        # You would need to type Yes to approve.

# Now delete the workspace.

terraform -chdir=aws workspace select default # change to default workspace
terraform -chdir=aws workspace delete <NAME_OF_WORKSPACE_THAT_IS_NOT_DEFAULT>  # delete workspace.
```

---

5. test the deployment
```bash
go-deploy --workspace REPLACE_ME_WITH_TERRAFORM_BACKEND --working-directory aws -verbose -dry-run --conf config-instance.yaml
```

7. deploy if all looks good.
```bash
go-deploy --workspace REPLACE_ME_WITH_TERRAFORM_BACKEND --working-directory aws -verbose --conf config-instance.yaml
# display the terraform state. The aws resources that were created.
go-deploy --workspace REPLACE_ME_WITH_TERRAFORM_BACKEND --working-directory aws -verbose -show
# display the public ip address of the aws instance
go-deploy --workspace REPLACE_ME_WITH_TERRAFORM_BACKEND --working-directory aws -verbose -output
```

Useful Details for troubleshooting:
This will produce an IP address in the resulting inventory.json file.
The previous command creates a terraform tfvars. These variables override the variables in `aws/main.tf`

**NOTE**: write down the IP address of the AWS instance that is created.

This can be found in `REPLACE_ME_WITH_TERRAFORM_BACKEND.cfg`  (e.g. production-YYYY-MM-DD.cfg, sm-test-go-fastapi-alias.cfg)
If you need to check what you have just done, here are some helpful Terraform commands:

```bash
cat REPLACE_ME_WITH_TERRAFORM_BACKEND.tfvars.json # e.g, production-YYYY-MM-DD.tfvars.json, sm-test-go-fastapi-alias.tfvars.json
```

The previous command creates an ansible inventory file.
```bash
cat REPLACE_ME_WITH_TERRAFORM_BACKEND-inventory.cfg  # e.g, production-YYYY-MM-DD-inventory, sm-test-go-fastapi-alias-inventory
```

Useful terraform commands to check what you have just done

```bash
terraform -chdir=aws workspace show   # current terraform workspace
terraform -chdir=aws show             # current state deployed ...
terraform -chdir=aws output           # shows public ip of aws instance 
```

## Configuring and deploying software (go-fastapi) _stack_:
These commands continue to be run in the dockerized development environment.

* Make DNS names for go-fastapi point to the public IP address. If using cloudflare, put the ip in cloudflare DNS record. Otherwise put the ip in the AWS Route 53 DNS record. 
* Location of SSH keys may need to be replaced after copying config-stack.yaml.sample
* s3 credentials are placed in a file using the format described above
* s3 uri if SSL is enabled. Location of SSL certs/key
* QoS mitigation if QoS is enabled
* Use the same workspace name as in the previous step

```bash
cp ./production/config-stack.yaml.sample ./config-stack.yaml
emacs ./config-stack.yaml    # MAKE SURE TO CHANGE THE GO-FASTAPI TAG (strip the v), also replace all the REPLACE_MEs
export ANSIBLE_HOST_KEY_CHECKING=False
````

**NOTE**: change the command below to point to the terraform workspace you use above. 
go-deploy --workspace REPLACE_ME_WITH_TERRAFORM_BACKEND --working-directory aws -verbose --conf config-stack.yaml

```bash
go-deploy --workspace REPLACE_ME_WITH_TERRAFORM_BACKEND --working-directory aws -verbose --conf config-stack.yaml
```


## Testing deployment:

1. Access go-fastapi from the command line by ssh'ing into the newly provisioned EC2 instance (this too is run via the dockerized dev environment):
ssh -i /tmp/go-ssh ubuntu@IP_ADDRESS

2. Access go-fastapi from a browser:

We use health checks in the `docker-compose` file.  
Use go-fastapi DNS name. http://{go-fastapi_host}/docs

3. Debugging:

* Use -dry-run and copy and paste the command and execute it manually
* ssh to the machine; the username is ubuntu. Try using DNS names to make sure they are fine.

```bash
docker-compose -f stage_dir/docker-compose.yaml ps
docker-compose -f stage_dir/docker-compose.yaml down # whenever you make any changes 
docker-compose -f stage_dir/docker-compose.yaml up -d
docker-compose -f stage_dir/docker-compose.yaml logs -f 
```

4. Testing LogRotate:

```bash
docker exec -u 0 -it apache_fastapi bash # enter the container
cat /opt/credentials/s3cfg

echo $S3_BUCKET
aws s3 ls s3://$S3_BUCKET
logrotate -v -f /etc/logrotate.d/apache2 # Use -f option to force log rotation.
cat /tmp/logrotate-to-s3.log # make sure uploading to s3 was fine
```

5. Testing Health Check:

```sh
docker inspect --format "{{json .State.Health }}" go-fastapi
```



### Helpful commands for the docker container used as a development environment (go-dev):

1. start the docker container `go-dev` in interactive mode.

```bash
docker run --rm --name go-dev -it geneontology/go-devops-base:tools-jammy-0.4.2  /bin/bash
```

In the command above we used the `--rm` option which means the container will be deleted when you exit.
If that is not the intent and you want to delete it later at your own convenience. Use the following `docker run` command.

```bash
docker run --name go-dev -it geneontology/go-devops-base:tools-jammy-0.4.2  /bin/bash
```

2. To exit or stop the container:

```bash
docker stop go-dev                   # stop container with the intent of restarting it. This is equivalent to `exit` inside the container.
docker start -ia go-dev              # restart and attach to the container.
docker rm -f go-dev                  # remove it for good.
```

3. Use `docker cp` to copy these credentials to /tmp:

```bash
docker cp /tmp/go-aws-credentials go-dev:/tmp/
docker cp /tmp/go-ssh go-dev:/tmp
docker cp /tmp/go-ssh.pub go-dev:/tmp
```

within the docker image:

```bash
chown root /tmp/go-*
chgrp root /tmp/go-*
chmod 400 /tmp/go-ssh
```
