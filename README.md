# BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM


In this project, we will utilize Terraform to simplify the process of setting up EKS, a managed Kubernetes service on AWS. With EKS, there is no need to manually install, manage, and maintain your own Kubernetes control plane. Furthermore, we will later employ Helm, a Kubernetes package manager, to deploy multiple applications efficiently.

## Configuring The Terraform Module For EKS

- Use AWS CLI to create an S3 bucket to store the Terraform state:`aws s3 mb s3://eks-bucket-backend-terraform-blami`

![x1](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/0eb00cb4-cf65-4a8b-98be-25021f3482b6)

Create a file – backend.tf
```
terraform {
  backend "s3" {
    bucket = "eks-bucket-backend-terraform-blami"
    key    = "terraform.tfstate"
    region = "us-east-2"
  }
}
```
Create a file – network.tf and provision Elastic IP for Nat Gateway, VPC, Private and public subnets.
```
# reserve Elastic IP to be used in our NAT gateway
resource "aws_eip" "nat_gw_elastic_ip" {
vpc = true

tags = {
Name            = "${var.cluster_name}-nat-eip"
iac_environment = var.iac_environment_tag
}
}

## Create VPC using the official AWS module

module "vpc" {
source  = "terraform-aws-modules/vpc/aws"

name = "${var.name_prefix}-vpc"
cidr = var.main_network_block
azs  = data.aws_availability_zones.available_azs.names

private_subnets = [
# this loop will create a one-line list as ["10.0.0.0/20", "10.0.16.0/20", "10.0.32.0/20", ...]
# with a length depending on how many Zones are available
for zone_id in data.aws_availability_zones.available_azs.zone_ids :
cidrsubnet(var.main_network_block, var.subnet_prefix_extension, tonumber(substr(zone_id, length(zone_id) - 1, 1)) - 1)
]

public_subnets = [
# this loop will create a one-line list as ["10.0.128.0/20", "10.0.144.0/20", "10.0.160.0/20", ...]
# with a length depending on how many Zones are available
# there is a zone Offset variable, to make sure no collisions are present with private subnet blocks
for zone_id in data.aws_availability_zones.available_azs.zone_ids :
cidrsubnet(var.main_network_block, var.subnet_prefix_extension, tonumber(substr(zone_id, length(zone_id) - 1, 1)) + var.zone_offset - 1)
]

# Enable single NAT Gateway to save some money
# WARNING: this could create a single point of failure, since we are creating a NAT Gateway in one AZ only
# feel free to change these options if you need to ensure full Availability without the need of running 'terraform apply'
# reference: https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/2.44.0#nat-gateway-scenarios
enable_nat_gateway     = true
single_nat_gateway     = true
one_nat_gateway_per_az = false
enable_dns_hostnames   = true
reuse_nat_ips          = true
external_nat_ip_ids    = [aws_eip.nat_gw_elastic_ip.id]

# Add VPC/Subnet tags required by EKS
tags = {
"kubernetes.io/cluster/${var.cluster_name}" = "shared"
iac_environment                             = var.iac_environment_tag
}
public_subnet_tags = {
"kubernetes.io/cluster/${var.cluster_name}" = "shared"
"kubernetes.io/role/elb"                    = "1"
iac_environment                             = var.iac_environment_tag
}
private_subnet_tags = {
"kubernetes.io/cluster/${var.cluster_name}" = "shared"
"kubernetes.io/role/internal-elb"           = "1"
iac_environment                             = var.iac_environment_tag
}
}

```
- It is crucial to add tags to the subnets in this project. The tags play a significant role in enabling the Kubernetes Cloud Controller Manager (cloud-controller-manager) and the AWS Load Balancer Controller (aws-load-balancer-controller) to identify the cluster's subnets. These controllers query the subnets using the tags as a filter to accurately locate and interact with the cluster.

## Create a file – variables.tf

```
# create some variables
variable "cluster_name" {
type        = string
description = "EKS cluster name."
}
variable "iac_environment_tag" {
type        = string
description = "AWS tag to indicate environment name of each infrastructure object."
}
variable "name_prefix" {
type        = string
description = "Prefix to be used on each infrastructure object Name created in AWS."
}
variable "main_network_block" {
type        = string
description = "Base CIDR block to be used in our VPC."
}
variable "subnet_prefix_extension" {
type        = number
description = "CIDR block bits extension to calculate CIDR blocks of each subnetwork."
}
variable "zone_offset" {
type        = number
description = "CIDR block bits extension offset to calculate Public subnets, avoiding collisions with Private subnets."
}

# create some variables
variable "admin_users" {
  type        = list(string)
  description = "List of Kubernetes admins."
}
variable "developer_users" {
  type        = list(string)
  description = "List of Kubernetes developers."
}
variable "asg_instance_types" {
  description = "List of EC2 instance machine types to be used in EKS."
}
variable "autoscaling_minimum_size_by_az" {
  type        = number
  description = "Minimum number of EC2 instances to autoscale our EKS cluster on each AZ."
}
variable "autoscaling_maximum_size_by_az" {
  type        = number
  description = "Maximum number of EC2 instances to autoscale our EKS cluster on each AZ."
}
variable "autoscaling_average_cpu" {
  type        = number
  description = "autoscaling_average_cpu"
```

## Create a file – data.tf

```
# get all available AZs in our region
data "aws_availability_zones" "available_azs" {
  state = "available"
  exclude_names = ["us-east-1e"]
}
data "aws_caller_identity" "current" {} # used for accesing Account ID and ARN
```

## Create a file – eks.tf

```
module "eks_cluster" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 18.0"
  cluster_name    = var.cluster_name
  cluster_version = "1.22"
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  cluster_endpoint_private_access = true
  cluster_endpoint_public_access = true

  # Self Managed Node Group(s)
  self_managed_node_group_defaults = {
    instance_type                          = var.asg_instance_types[0]
    update_launch_template_default_version = true
  }
  self_managed_node_groups = local.self_managed_node_groups

  # aws-auth configmap
  create_aws_auth_configmap = true
  manage_aws_auth_configmap = true
  aws_auth_users = concat(local.admin_user_map_users, local.developer_user_map_users)
  tags = {
    Environment = "prod"
    Terraform   = "true"
  }
}
```

- In Terraform, it is not possible to directly assign a variable to another variable. This design choice is intentional to prevent unnecessary code duplication. Instead, the recommended approach in Terraform is to utilize locals. By using locals, you can maintain the DRY (Don't Repeat Yourself) principle and avoid repetitive code. Locals allow you to define reusable expressions and calculations within your Terraform configuration, making it easier to manage and maintain your codebase.

## Create a file – locals.tf

```
# render Admin & Developer users list with the structure required by EKS module
locals {
  admin_user_map_users = [
    for admin_user in var.admin_users :
    {
      userarn  = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:user/${admin_user}"
      username = admin_user
      groups   = ["system:masters"]
    }
  ]
  developer_user_map_users = [
    for developer_user in var.developer_users :
    {
      userarn  = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:user/${developer_user}"
      username = developer_user
      groups   = ["${var.name_prefix}-developers"]
    }
  ]

  self_managed_node_groups = {
    worker_group1 = {
      name = "${var.cluster_name}-wg"

      min_size      = var.autoscaling_minimum_size_by_az * length(data.aws_availability_zones.available_azs.zone_ids)
      desired_size      = var.autoscaling_minimum_size_by_az * length(data.aws_availability_zones.available_azs.zone_ids)
      max_size  = var.autoscaling_maximum_size_by_az * length(data.aws_availability_zones.available_azs.zone_ids)
      instance_type = var.asg_instance_types[0].instance_type

      bootstrap_extra_args = "--kubelet-extra-args '--node-labels=node.kubernetes.io/lifecycle=spot'"

      block_device_mappings = {
        xvda = {
          device_name = "/dev/xvda"
          ebs = {
            delete_on_termination = true
            encrypted             = false
            volume_size           = 10
            volume_type           = "gp2"
          }
        }
      }

      use_mixed_instances_policy = true
      mixed_instances_policy = {
        instances_distribution = {
          spot_instance_pools = 4
        }

        override = var.asg_instance_types
      }
    }
  }
}

```
## Create a file – variables.tfvars to set values for variables.

```
cluster_name            = "tooling-app-eks"
iac_environment_tag     = "development"
name_prefix             = "darey-io-eks"
main_network_block      = "10.0.0.0/16"
subnet_prefix_extension = 4
zone_offset             = 8

# Ensure that these users already exist in AWS IAM. Another approach is that you can introduce an iam.tf file to manage users separately, get the data source and interpolate their ARN.
admin_users                    = ["kubernetes"]
developer_users                = ["kubernetes"]
asg_instance_types             = [ { instance_type = "t3.small" }, { instance_type = "t2.small" }, ]
autoscaling_minimum_size_by_az = 1
autoscaling_maximum_size_by_az = 10
autoscaling_average_cpu                  = 30
```

- Run `terraform init`

![x2](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/f6a0dfaf-3a70-4a97-a481-3c4930f9a549)

- Run `terraform plan` 

![x3](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/0377b55a-b008-49ab-a3cb-b14681f336b5)

- Run `terraform apply`. When executing the "terraform apply" command to create resources, it is likely that you will encounter an error at some point during the process. This error typically arises due to Terraform's need to establish a connection and accurately configure the credentials in order to connect to the cluster using the kubeconfig.
- 
![x4](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/457ded41-e9c0-49ff-9f5d-c18f93c94cb7)

- To fix this problem , Append the file data.tf file

# get EKS cluster info to configure Kubernetes and Helm providers
```
data "aws_eks_cluster" "cluster" {
  name = module.eks_cluster.cluster_id
}
data "aws_eks_cluster_auth" "cluster" {
  name = module.eks_cluster.cluster_id
}

```

## Append to the file provider.tf

```
# get EKS authentication for being able to manage k8s objects from terraform
provider "kubernetes" {
  host                   = data.aws_eks_cluster.cluster.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority.0.data)
  token                  = data.aws_eks_cluster_auth.cluster.token
}

```

- Run the init and plan again

![x5](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/a7e5a092-9774-47fa-bc90-32e132b651a6)

![x6](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/9221d7bb-7ff1-4a3f-bf27-111a25ebacae)

- Create kubeconfig file using awscli.

![x7](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/be7a9ea0-f56b-4398-93f9-eed1d2f15fe7)

## Installing Helm From Script

- `wget https://github.com/helm/helm/archive/refs/tags/v3.6.3.tar.gz`
- Unpack the tar.gz file. ` tar -zxvf v3.6.3.tar.gz` 
- `cd helm-3.6.3`
- Build the source code using make utility. `make build`
- `sudo mv bin/helm /usr/bin/`
- Check that Helm is installed

![x8](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/5fa685ec-9320-4c2c-8458-1eebbad2bb2b)

## DEPLOY JENKINS WITH HELM

- One of the remarkable aspects of Helm is its ability to deploy applications easily by leveraging pre-packaged charts from public Helm repositories. This means that you can deploy applications like Jenkins with minimal configuration required. Helm simplifies the process by providing a convenient way to fetch the necessary dependencies and deploy applications with just a few simple commands. For instance, you can quickly deploy Jenkins by specifying the appropriate Helm chart and providing any necessary customizations or configuration options. This streamlined approach saves time and effort, making application deployment a breeze.
 
 ## STEPS 
 
- Visit Artifact Hub to find packaged applications as Helm Charts
- Search for Jenkins
- Add the repository to helm so that you can easily download and deploy. `helm repo update`
- `helm install myjenkins jenkins/jenkins --kubeconfig kubeconfig file`

![x19](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/912b8978-01a0-4fe9-83f0-983887ce86e2)

- Check the Helm deployment `helm ls --kubeconfig kubeconfig file`
- 
![x45](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/078a72e8-a12e-4d3f-9cd6-5ae8a0cbb633)

- Check the pods

![x21](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/a4443d19-dcf5-4e89-a5bc-fb24e74483b5)

- Describe the running pod

![x22](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/934287e6-0198-4af2-912f-1f9be7b3031f)

- Check the logs of the running pod, You will notice an output with an error.

![x46](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/ccaee3b2-e949-48da-b650-2e9936807041)

- Since the pod contains multiple containers, including the Jenkins container and the config-reload Sidecar container, it is necessary to specify the exact pod for which you want to view the logs. This additional information will help kubectl identify the specific pod you are interested in. Consequently, the command should be updated to include the pod name or identifier, ensuring that kubectl retrieves the logs for the correct pod.Hence, the command will be updated like`kubectl logs myjenkins-0 -c jenkins --kubeconfig kubeconfig file`

- To avoid the need to call the kubeconfig file every time and to handle situations where the default kubeconfig file may already be in use by another cluster, you can leverage a kubectl plugin called "konfig". This plugin allows you to merge multiple kubeconfig files together, providing a unified configuration. You can then select the desired kubeconfig to be active.To install the krew package manager for kubectl and enable the installation of plugins to enhance the functionality of kubectl, follow these steps:
  - Install the konfig plugin `kubectl krew install konfig`
  - Import the kubeconfig into the default kubeconfig file. Ensure to accept the prompt to overide.`sudo kubectl konfig import --save  kubeconfig file`
  - Show all the contexts – Meaning all the clusters configured in your kubeconfig. If you have more than 1 Kubernetes clusters configured, you will see them all     in the output. `kubectl config get-contexts`
  
  ![x26](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/93907242-bc3e-4acf-a06e-654803b1ef09)

  - Set the current context to use for all kubectl and helm commands `kubectl config use-context tooling-app-eks`
  
  ![x23](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/6dabcc51-576d-4847-abe5-9fec50aeadf0)

  - Test that it is working without specifying the --kubeconfig flag `kubectl get po`

    ![x28](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/0db7f694-461d-41d5-8e1f-908888a212ce)

  - Display the current context. This will let you know the context in which you are using to interact with Kubernetes.`kubectl config current-context`
   
  ![x27](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/a7314a0c-e5ae-4e15-9a35-877bd4528067)

  
- Now that we can use kubectl without the --kubeconfig flag, Lets get access to the Jenkins UI.
- There are some commands that was provided on the screen when Jenkins was installed with Helm.Run the commands to set up Jenkins.
- Get the password to the admin user `kubectl exec --namespace default -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/chart-admin-password && echo`
- Use port forwarding to access Jenkins from the UI `kubectl --namespace default port-forward svc/jenkins 8080:8080`

![x29](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/9efcd472-82cb-4919-9d4e-dd50bf758e7c)

![x30](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/51470020-58fd-4670-8b73-3b27a79a0c57)

![x31](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/0e10899e-1dcb-47e2-9afd-9034226394f3)

- To install Prometheus, Nginx, and Grafana, you can visit the ArtifactHub website and follow the instructions provided to download the respective Helm charts.
  - Access the ArtifactHub website.
  - Search for the desired charts, such as "Prometheus," "Nginx," and "Grafana."
  - Locate the Helm charts for each application and click on their respective links.
  - On the chart page, you will find installation instructions specific to each chart.
  - Follow the provided instructions to download and install the Helm charts using the Helm package manager.
  
  ![x39](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/1e3ec760-27f5-4622-839c-01df6ccb58c0)
  
  ![x40](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/e13210b5-3093-4561-a1ab-01354da40e37)
  
  ![x41](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/31adfe5b-fc41-4ce0-83bb-77dcee1d3018)
  
  ![x43](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/fd3c0815-2192-4493-bb8f-5ab5a9286401)
  
  ![x42](https://github.com/busolagbadero/BUILDING-ELASTIC-KUBERNETES-SERVICE-EKS-WITH-TERRAFORM/assets/94229949/bbfccd24-980f-4b98-999c-fb847fc13b9d)

  





