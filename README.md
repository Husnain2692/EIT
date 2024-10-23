markdown
Copy code
# Kubernetes Clusters on AWS EKS Using Terraform

This project demonstrates how to deploy a simple Dockerized HTML page on an AWS Elastic Kubernetes Service (EKS) cluster using Terraform. We walk through setting up the infrastructure, deploying the HTML application, and exposing it to the internet.

## Project Structure

```bash
terraform-eks/
├── main.tf         # Main Terraform configuration for AWS and EKS setup
├── kubernetes.tf   # Kubernetes resource configurations for the cluster
├── variables.tf    # Variables for AWS region, VPC, and subnets
├── outputs.tf      # Outputs for EKS cluster endpoint and other resources
├── html/
│   └── index.html  # Static HTML page to be served
│   └── Dockerfile  # Dockerfile for building the HTML page container
Steps to Deploy
Step 1: Create variables.tf
Define variables for AWS region, VPC, and subnets:

hcl
Copy code
variable "aws_region" {
  description = "The AWS region to create resources in"
  default     = "us-east-1"
}

variable "aws_vpc" {
  description = "The default VPC in AWS account"
  default     = "vpc-xxx"
}

variable "vpc_subnet_1" {
  description = "The first subnet"
  default     = "subnet-xxx"
}

variable "vpc_subnet_2" {
  description = "The second subnet"
  default     = "subnet-xxx"
}
Step 2: Create main.tf
Set up the AWS provider and EKS cluster using the Terraform AWS EKS module:

hcl
Copy code
# AWS Provider Configuration
provider "aws" {
  region = var.aws_region
}

# EKS Cluster Setup
module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "simple-eks-cluster"
  cluster_version = "1.30"
  vpc_id          = var.aws_vpc
  subnet_ids      = [var.vpc_subnet_1, var.vpc_subnet_2]

  eks_managed_node_groups = {
    eks_nodes = {
      desired_capacity = 2
      max_capacity     = 2
      min_capacity     = 1
      instance_type    = "t3.micro"
    }
  }
}
Step 3: Create index.html and Dockerfile
index.html:
html
Copy code
<!DOCTYPE html>
<html>
  <head>
    <title>Welcome to My EKS App</title>
  </head>
  <body>
    <h1>Hello from AWS EKS Cluster!</h1>
  </body>
</html>
Dockerfile:
Dockerfile
Copy code
FROM nginx:alpine
COPY ./index.html /usr/share/nginx/html/index.html
Build and push the container image to Docker Hub:

bash
Copy code
docker build -t <dockerhub-username>/html-app:latest .
docker push <dockerhub-username>/html-app:latest
Step 4: Create kubernetes.tf
Configure Kubernetes resources, such as the namespace, deployment, and service:

hcl
Copy code
# Kubernetes Provider Configuration
provider "kubernetes" {
  host                   = module.eks.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
  token                  = data.aws_eks_cluster_auth.my_cluster.token
}

data "aws_eks_cluster_auth" "my_cluster" {
  name = module.eks.cluster_name
}

# Kubernetes Namespace
resource "kubernetes_namespace" "app_namespace" {
  metadata {
    name = "html-app"
  }
}

# HTML App Deployment
resource "kubernetes_deployment" "html_app" {
  metadata {
    name      = "html-deployment"
    namespace = kubernetes_namespace.app_namespace.metadata[0].name
  }

  spec {
    replicas = 2
    selector {
      match_labels = { app = "html-app" }
    }

    template {
      metadata { labels = { app = "html-app" } }
      spec {
        container {
          image = "<dockerhub-username>/html-app:latest"
          name  = "html-app"
          port  { container_port = 80 }
        }
      }
    }
  }
}

# Service to expose HTML App
resource "kubernetes_service" "html_app_service" {
  metadata { name = "html-app-service"; namespace = kubernetes_namespace.app_namespace.metadata[0].name }

  spec {
    selector = { app = "html-app" }
    port { port = 80; target_port = 80 }
    type = "LoadBalancer"
  }
}
Step 5: Run Terraform Commands
Initialize Terraform:

bash
Copy code
terraform init
Apply the configuration:

bash
Copy code
terraform apply -auto-approve
Step 6: Verification
Configure kubectl to access the EKS cluster:

bash
Copy code
aws eks --region <aws_region> update-kubeconfig --name <eks_cluster_name>
Get the external IP of the LoadBalancer:

bash
Copy code
kubectl get svc -n html-app
