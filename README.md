# Kubernetes not the hardest way (or "Provisioning a Kubernetes Cluster on AWS using Terraform and Ansible")

> [!NOTE]
> This repo has been archived and is no longer under development. OpenCredo is now part of [Trifork](https://github.com/trifork).

A worked example to provision a Kubernetes cluster on AWS from scratch, using Terraform and Ansible. A scripted version of the famous tutorial [Kubernetes the hard way](https://github.com/kelseyhightower/kubernetes-the-hard-way).

See the companion article https://opencredo.com/kubernetes-aws-terraform-ansible-1/ for details about goals, design decisions and simplifications.

- AWS VPC
- 3 EC2 instances for HA Kubernetes Control Plane: Kubernetes API, Scheduler and Controller Manager
- 3 EC2 instances for *etcd* cluster
- 3 EC2 instances as Kubernetes Workers (aka Minions or Nodes)
- Kubenet Pod networking (using CNI)
- HTTPS between components and control API
- Sample *nginx* service deployed to check everything works

*This is a learning tool, not a production-ready setup.*

## Requirements

Requirements on control machine:

- Terraform (tested with Terraform 0.7.0; **NOT compatible with Terraform 0.6.x**)
- Python (tested with Python 2.7.12, may be not compatible with older versions; requires Jinja2 2.8)
- Python *netaddr* module
- Ansible (tested with Ansible 2.1.0.0)
- *cfssl* and *cfssljson*:  https://github.com/cloudflare/cfssl
- Kubernetes CLI
- SSH Agent
- (optionally) AWS CLI


## AWS Credentials

### AWS KeyPair

You need a valid AWS Identity (`.pem`) file and the corresponding Public Key. Terraform imports the [KeyPair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html) in your AWS account. Ansible uses the Identity to SSH into machines.

Please read [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#how-to-generate-your-own-key-and-import-it-to-aws) about supported formats.

### Terraform and Ansible authentication

Both Terraform and Ansible expect AWS credentials set in environment variables:
```
$ export AWS_ACCESS_KEY_ID=<access-key-id>
$ export AWS_SECRET_ACCESS_KEY="<secret-key>"
```

If you plan to use AWS CLI you have to set `AWS_DEFAULT_REGION`.

Ansible expects the SSH identity loaded by SSH agent:
```
$ ssh-add <keypair-name>.pem
```

## Defining the environment

Terraform expects some variables to define your working environment:

- `control_cidr`: The CIDR of your IP. All instances will accept only traffic from this address only. Note this is a CIDR, not a single IP. e.g. `123.45.67.89/32` (mandatory)
- `default_keypair_public_key`: Valid public key corresponding to the Identity you will use to SSH into VMs. e.g. `"ssh-rsa AAA....xyz"` (mandatory)

**Note that Instances and Kubernetes API will be accessible only from the "control IP"**. If you fail to set it correctly, you will not be able to SSH into machines or run Ansible playbooks.

You may optionally redefine:

- `default_keypair_name`: AWS key-pair name for all instances.  (Default: "k8s-not-the-hardest-way")
- `vpc_name`: VPC Name. Must be unique in the AWS Account (Default: "kubernetes")
- `elb_name`: ELB Name for Kubernetes API. Can only contain characters valid for DNS names. Must be unique in the AWS Account (Default: "kubernetes")
- `owner`: `Owner` tag added to all AWS resources. No functional use. It becomes useful to filter your resources on AWS console if you are sharing the same AWS account with others. (Default: "kubernetes")



The easiest way is creating a `terraform.tfvars` [variable file](https://www.terraform.io/docs/configuration/variables.html#variable-files) in `./terraform` directory. Terraform automatically imports it.

Sample `terraform.tfvars`:
```
default_keypair_public_key = "ssh-rsa AAA...zzz"
control_cidr = "123.45.67.89/32"
default_keypair_name = "lorenzo-glf"
vpc_name = "Lorenzo ETCD"
elb_name = "lorenzo-etcd"
owner = "Lorenzo"
```


### Changing AWS Region

By default, the project uses `eu-west-1`. To use a different AWS Region, set additional Terraform variables:

- `region`: AWS Region (default: "eu-west-1").
- `zone`: AWS Availability Zone (default: "eu-west-1a")
- `default_ami`: Pick the AMI for the new Region from https://cloud-images.ubuntu.com/locator/ec2/: Ubuntu 16.04 LTS (xenial), HVM:EBS-SSD

You also have to edit `./ansible/hosts/ec2.ini`, changing `regions = eu-west-1` to the new Region.

## Provision infrastructure, with Terraform

Run Terraform commands from `./terraform` subdirectory.

```
$ terraform plan
$ terraform apply
```

Terraform outputs public DNS name of Kubernetes API and Workers public IPs.
```
Apply complete! Resources: 12 added, 2 changed, 0 destroyed.
  ...
Outputs:

  kubernetes_api_dns_name = lorenzo-kubernetes-api-elb-1566716572.eu-west-1.elb.amazonaws.com
  kubernetes_workers_public_ip = 54.171.180.238,54.229.249.240,54.229.251.124
```

You will need them later (you may show them at any moment with `terraform output`).

### Generated SSH config

Terraform generates `ssh.cfg`, SSH configuration file in the project directory.
It is convenient for manually SSH into machines using node names (`controller0`...`controller2`, `etcd0`...`2`, `worker0`...`2`), but it is NOT used by Ansible.

e.g.
```
$ ssh -F ssh.cfg worker0
```

## Install Kubernetes, with Ansible

Run Ansible commands from `./ansible` subdirectory.

We have multiple playbooks.

### Install and set up Kubernetes cluster

Install Kubernetes components and *etcd* cluster.
```
$ ansible-playbook infra.yaml
```

### Setup Kubernetes CLI

Configure Kubernetes CLI (`kubectl`) on your machine, setting Kubernetes API endpoint (as returned by Terraform).
```
$ ansible-playbook kubectl.yaml --extra-vars "kubernetes_api_endpoint=<kubernetes-api-dns-name>"
```

Verify all components and minions (workers) are up and running, using Kubernetes CLI (`kubectl`).

```
$ kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}

$ kubectl get nodes
NAME                                       STATUS    AGE
ip-10-43-0-30.eu-west-1.compute.internal   Ready     6m
ip-10-43-0-31.eu-west-1.compute.internal   Ready     6m
ip-10-43-0-32.eu-west-1.compute.internal   Ready     6m
```

### Setup Pod cluster routing

Set up additional routes for traffic between Pods.
```
$ ansible-playbook kubernetes-routing.yaml
```

### Smoke test: Deploy *nginx* service

Deploy a *ngnix* service inside Kubernetes.
```
$ ansible-playbook kubernetes-nginx.yaml
```

Verify pods and service are up and running.

```
$ kubectl get pods -o wide
NAME                     READY     STATUS    RESTARTS   AGE       IP           NODE
nginx-2032906785-9chju   1/1       Running   0          3m        10.200.1.2   ip-10-43-0-31.eu-west-1.compute.internal
nginx-2032906785-anu2z   1/1       Running   0          3m        10.200.2.3   ip-10-43-0-30.eu-west-1.compute.internal
nginx-2032906785-ynuhi   1/1       Running   0          3m        10.200.0.3   ip-10-43-0-32.eu-west-1.compute.internal

> kubectl get svc nginx --output=json
{
    "kind": "Service",
    "apiVersion": "v1",
    "metadata": {
        "name": "nginx",
        "namespace": "default",
...
```

Retrieve the port *nginx* has been exposed on:

```
$ kubectl get svc nginx --output=jsonpath='{range .spec.ports[0]}{.nodePort}'
32700
```

Now you should be able to access *nginx* default page:
```
$ curl http://<worker-0-public-ip>:<exposed-port>
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

The service is exposed on all Workers using the same port (see Workers public IPs in Terraform output).


# Known simplifications

There are many known simplifications, compared to a production-ready solution:

- Networking setup is very simple: ALL instances have a public IP (though only accessible from a configurable Control IP).
- Infrastructure managed by direct SSH into instances (no VPN, no Bastion).
- Very basic Service Account and Secret (to change them, modify: `./ansible/roles/controller/files/token.csv` and `./ansible/roles/worker/templates/kubeconfig.j2`)
- No actual integration between Kubernetes and AWS.
- No additional Kubernetes add-on (DNS, Dashboard, Logging...)
- Simplified Ansible lifecycle. Playbooks support changes in a simplistic way, including possibly unnecessary restarts.
- Instances use static private IP addresses
- No stable private or public DNS naming (only dynamic DNS names, generated by AWS)
