# Udacity Capstone Infra

This repository provides method to create underlying Kubernetes infrastructure for running containerised applications. This step is needed to sucesfully complete Cloud DevOps Engineer Nanodegree Program Capstone project. The simpliest way would be to use official CLI for Amazon EKS - `eksctl`.

## Prerequisites

You will need:

* an AWS account with the IAM permissions listed on the EKS module documentation,
* a configured AWS CLI
* kubectl
* direnv
  
## Configuration files

### AWS Authentication

We are using Environment Variables for providing credentials for authentication. Template file for applying AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY can be found in `.envrc-dist` file.

Copy `.envrc-dist` to `.envrc` add the exact parameters 

* replace <AWS_ACCESS_KEY_ID> with your Access key ID
* replace <AWS_SECRET_ACCESS_KEY> with Secret access key

and let `direnv`  load environment variables.

## Set up and initialize your Terraform workspace

In your terminal, clone the following repository.

```bash
$ git clone https://github.com/loncarales/udacity-capstone-infra.git
```

Copy `main.tf-dist` file to `main.tf` file and configure the following:

```JSON
backend "s3" {
    bucket         = "<S3_BUCKET_NAME>" // S3 bucket name
    key            = "global/s3/<OBJECT_KEY>.tfstate" // Object key name
    region         = "<AWS_REGION>" // AWS Region
    dynamodb_table = "<DynamoDB_TABLE_NAME>" // DynamoDB table name
    encrypt        = true
}
```

The values must be the same as configured in the repository [https://github.com/loncarales/terraform-state-management](https://github.com/loncarales/terraform-state-management).


### Initialize Terraform workspace

Once you have cloned the repository, initialize your Terraform workspace, which will download and configure the providers.

```bash
$ terraform init
$ terraform plan
```

### Provision the EKS cluster

Run `terraform apply` and review the planned actions.

Your terminal output should indicate the plan is running and what resources will be created. This process should take approximately 10 - 15 minutes. Upon successful application, your terminal prints the outputs defined in `outputs.tf`.

## Configure kubectl

Run the following command to retrieve the access credentials for your cluster and automatically configure `kubectl`.

```bash
$ aws eks --region $(terraform output -raw region) update-kubeconfig --name $(terraform output -raw cluster_name)
```

## Deploy and access Kubernetes Dashboard

### Deploy Kubernetes Metrics Server

While we can deploy the Kubernetes metrics server and dashboard using Terraform, kubectl is used so we don't need to configure our Terraform Kubernetes Provider.

Download and unzip the metrics server by running the following command.

```bash
$ wget -O v0.3.6.tar.gz https://codeload.github.com/kubernetes-sigs/metrics-server/tar.gz/v0.3.6 && tar -xzf v0.3.6.tar.gz
$ kubectl apply -f metrics-server-0.3.6/deploy/1.8+/
$ kubectl get deployment metrics-server -n kube-system
```

### Deploy Kubernetes Dashboard

The following command will schedule the resources necessary for the dashboard.

```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
```

Now, create a proxy server that will allow you to navigate to the dashboard from the browser on your local machine. This will continue running until you stop the process by pressing `CTRL + C`.

```bash
$ kubectl proxy 
```

You should be able to access the Kubernetes dashboard [here](http://127.0.0.1:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/).

### Authenticate the dashboard

To use the Kubernetes dashboard, you need to create a `ClusterRoleBinding` and provide an authorization token. This gives the `cluster-admin` permission to access the `kubernetes-dashboard`. Authenticating using kubeconfig is **not** an option.

In another terminal (do not close the kubectl proxy process), create the ClusterRoleBinding resource.

```bash
$ kubectl apply -f https://raw.githubusercontent.com/hashicorp/learn-terraform-provision-eks-cluster/master/kubernetes-dashboard-admin.rbac.yaml
```

Then, generate the authorization token.

```bash
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep service-controller-token | awk '{print $1}')
```

## Clean up your workspace

Remember to destroy any resources you create once you no longer need them. Run the destroy command and confirm with `yes` in your terminal.

```bash
$ terraform destroy
```
