Kubernetes Clusters on AWS EKS Using Terraform
In this project, we will demonstrate how to deploy a simple dockerized HTML page on an AWS Elastic Kubernetes Service (EKS) cluster using Terraform. We will walk through the steps of setting up the infrastructure, deploying the HTML application, and exposing it to the internet. The source code for this project is available in the following GitHub repository: GitHub Repository Link.
Project Structure
The project directory is organized as follows:
graphql
Copy code
terraform-eks/
├── main.tf         # Main Terraform configuration for AWS and EKS setup
├── kubernetes.tf   # Kubernetes resource configurations for the cluster
├── variables.tf    # Variables for AWS region, VPC, and subnets
├── outputs.tf      # Outputs for EKS cluster endpoint and other resources
├── html/
│   └── index.html  # Static HTML page to be served
│   └── Dockerfile  # Dockerfile for building the HTML page container


Step 1: Create Variables.tf
The variables.tf file defines variables that allow for customization, including the AWS region, VPC, and subnets.
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




Step 2: Create Main.tf
The main.tf file is the core of the Terraform configuration. It sets up the AWS provider and creates the EKS cluster using the Terraform AWS EKS module.
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
The index.html file contains a simple static HTML page, while the Dockerfile sets up a container image to serve this page using Nginx.
Example index.html:
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
Example Dockerfile:
Dockerfile
Copy code
FROM nginx:alpine
COPY ./index.html /usr/share/nginx/html/index.html

Build and push the container image to Docker Hub or another container registry:
bash
Copy code
docker build -t <dockerhub-username>/html-app:latest .
docker push <dockerhub-username>/html-app:latest


Step 4: Create Kubernetes.tf
This file configures Kubernetes resources such as a namespace, a deployment for the HTML app, and a LoadBalancer service to expose it.
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
      match_labels = {
        app = "html-app"
      }
    }

    template {
      metadata {
        labels = {
          app = "html-app"
        }
      }

      spec {
        container {
          image = "<dockerhub-username>/html-app:latest"
          name  = "html-app"

          port {
            container_port = 80
          }
        }
      }
    }
  }
}

# Service for Exposing the HTML App
resource "kubernetes_service" "html_app_service" {
  metadata {
    name      = "html-app-service"
    namespace = kubernetes_namespace.app_namespace.metadata[0].name
  }

  spec {
    selector = {
      app = "html-app"
    }

    port {
      port        = 80
      target_port = 80
    }

    type = "LoadBalancer"
  }
}


Step 5: Terraform Commands
1. Initialize Terraform:
bash
Copy code
terraform init


2. Apply the configuration:
bash
Copy code
terraform apply -auto-approve

Terraform will create the necessary resources and set up the EKS cluster.


Step 6: Verification
1. Install and Configure kubectl:
To interact with the EKS cluster, install kubectl and configure it using the following command:
bash
Copy code
aws eks --region <aws_region> update-kubeconfig --name <eks_cluster_name>



2. Fetch the External IP of the LoadBalancer:
Once the cluster is up and running, retrieve the external IP of the LoadBalancer to access the HTML page in your browser:
bash
Copy code
kubectl get svc -n html-app

The external IP will allow you to view your static HTML page deployed on the EKS cluster.

Conclusion
By following this guide, you have successfully set up an AWS EKS cluster using Terraform and deployed a simple HTML app using Kubernetes. You have learned how to create, configure, and deploy a Kubernetes cluster using Terraform, making infrastructure management more efficient and scalable.
