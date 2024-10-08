## Experiments and random stuff

This repo contains some notes about setting up K8s cluster on EC2+ECR without EKS to see how fast it can be. EKS can take between 10 and 15 minutes to boostrap a cluster.

Below is a changelog of this repo.

### Day 1: Set up Terraform with a remote backend

Commit: https://github.com/lucabrunox/experiments/tree/cd8378154c378

Define the AWS region that will be used for all the commands:

```bash
export AWS_REGION=eu-west-1
```

Create an S3 bucket for terraform state:

```bash
aws s3api create-bucket --acl private --bucket experiments-12345-terraform --create-bucket-configuration LocationConstraint=$AWS_REGION
```

Initialize:

```bash
cd tf

cat <<EOF > backend.conf
bucket = "experiments-12345-terraform"
key    = "experiments/terraform.tfstate"
region = "$AWS_REGION"
EOF

terraform init --backend-config=backend.conf
terraform get
```

Apply the plan which will create a K8s cluster:

```bash
terraform apply \
  -var "region=$AWS_REGION" \
  -var "asg_desired_capacity=1" \
  -var "nlb_enabled=true"
```

Use asg_desired_capacity=0 to tear down the cluster.

### Day 2: Kubernetes single-node cluster on EC2 with kubeadm

Commit: https://github.com/lucabrunox/experiments/tree/9cc3ac81d7f

Using a raw K8s on EC2 instead of EKS, using Flannel instead of AWS CNI. Some interesting facts:

- It takes 2 minutes and 20 second until all containers are in Running state.
- A t4g.medium is needed to run a cluster. Using a t4g.nano with swap is not enough because the apiserver/etcd will keep timing out.

SSH into the EC2 instance and run crictl and kubectl commands to inspect the cluster:

```bash
sudo su

crictl ps

export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl get all -A
```

To check init logs:

```bash
sudo cat /var/log/cloud-init-output.log
```

### Day 3: Sample app with Docker pushed to ECR

Commit: https://github.com/lucabrunox/experiments/tree/5216dfe5efd6

Set up following https://docs.djangoproject.com/en/5.1/intro/tutorial01/ with `django-admin startproject mysite`.

The Dockerfile is self-explanatory:

```bash
docker run -p 8000:8000 --rm -it $(docker build -q .)
```

To test GH actions I've set up act to run in a Docker, which seems to work fine:

```bash
./test_gh.sh
```

### Day 4: Deploy the app in K8s using the ECR image

Commit: https://github.com/lucabrunox/experiments/tree/5216dfe5efd6

We're using a CronJob to get the ECR creds from the node metadata.

Some notes:
- Need to untaint control plane node in other to schedule pods.
- As we use t4g changed the GH build to ARM.
- Python requirements also need tzdata.

At the end we're able to execute the following kubectl on the EC2 instance to deploy the app and watch it working:

```bash
kubectl apply -f k8s/ecr-credentials.yaml
kubectl apply -f frontend/k8s/manifest.yaml

curl $(kubectl get svc frontend -o=jsonpath='{.spec.clusterIP}'):8000
```

### Day 5: Expose service via NLB and NodePort

Commit: https://github.com/lucabrunox/experiments/tree/f03d8449f869

Some notes:
- Configured NLB with the security group in 2 subnets.
- Django has an ALLOWED_HOSTS config to prevent Host header attacks
- Django detects a tty when logging to stdout

```bash
kubectl apply -f k8s/ecr-credentials.yaml
kubectl apply -f frontend/k8s/manifest.yaml

curl http://$(terraform output --raw experiments_nlb_dns_name)
```

### Day 6: Helm for the app

Commit: https://github.com/lucabrunox/experiments/tree/0427c9b777aaab

Planning to create multiple instances of the same app, so packaging it with helm.

Some notes:
- Created the helm with the default helm create scaffolding.
- Added container hostname to ALLOWED_HOSTS for the health checks.

```bash
kubectl apply -f k8s/ecr-credentials.yaml
helm install --set-string image=ACCOUNT_ID.dkr.ecr.eu-west-1.amazonaws.com/experiments-frontend:vTAG frontend ./frontend/k8s/chart

curl http://$(terraform output --raw experiments_nlb_dns_name)
```

## Day 7: Split into multiple files

Commit: https://github.com/lucabrunox/experiments/tree/c6db952745cd9f

Created a self-contained k8s_control_plane_template module. Obviously, it's only a single node.

For the rest I'm leaving them in the main module.

## Day 8: Easier way to access node and kubectl

Until now, we've been manually getting the EC2 IP and then using admin.conf as root. With this change instead we tag the k8s node instance, and give ec2-user admin access so that we can ssh and run kubectl commands straightaway.

An interesting note is that `kubectl certificate approve` has a delay, hence we must wait for the certificate to be ready first.

```bash
ssh ec2-user@$(aws ec2 describe-instances --filters "Name=instance-state-name,Values=running" --filters "Name=tag-key,Values=Name" "Name=tag-value,Values=experiments_k8s_control_plane" --query 'Reservations[].Instances[].PublicIpAddress' --output text)

kubectl get all -A
```
