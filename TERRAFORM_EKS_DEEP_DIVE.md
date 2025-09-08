# 🏗️ Terraform EKS Infrastructure Deep Dive

## 📋 Overview

This document provides a comprehensive analysis of the Terraform code used to create the sophisticated AWS EKS networking architecture in the doTERRA LeaderTools QAS environment.

---

## 🎯 Terraform Infrastructure Components

### **Core Infrastructure Files Structure:**
```
terraform/
├── eks.tf                    # Main EKS cluster configuration
├── eks-alb-ingress.tf       # AWS Load Balancer Controller
├── eks-karpenter.tf         # Karpenter auto-scaling
├── eks-auth.tf              # Authentication and authorization
├── variables.tf             # Input variables and validation
├── outputs.tf               # Output values
├── provider.tf              # AWS provider configuration
├── versions.tf              # Terraform and provider versions
└── environments/
    └── qas/
        ├── backend.hcl      # Remote state configuration
        └── variables.tfvars # Environment-specific values
```

---

## 🚀 EKS Cluster Core Configuration (`eks.tf`)

### **1. Terraform Modules Used:**
```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.20"
  
  # This is the official AWS EKS Terraform module
  # Version 20.20+ supports latest EKS features and Kubernetes 1.33
}
```

**Why This Module?**
- ✅ **Community maintained** - Most popular EKS module (100K+ downloads)
- ✅ **AWS best practices** - Follows official AWS recommendations
- ✅ **Regular updates** - Supports latest Kubernetes versions
- ✅ **Comprehensive features** - Handles complex networking scenarios

### **2. Cluster Basic Configuration:**
```hcl
cluster_name                    = "leadertools-qas"
cluster_version                 = "1.33"           # Latest Kubernetes version
cluster_endpoint_public_access  = true            # Public API access
cluster_endpoint_private_access = false           # No private API endpoint

# Modern authentication mode (hybrid approach)
authentication_mode                      = "API_AND_CONFIG_MAP"
enable_cluster_creator_admin_permissions = false  # Security best practice
```

**Key Decisions Explained:**
- **Kubernetes 1.33:** Latest stable version with newest features
- **Public API only:** Simplifies management, secured by IAM/OIDC
- **API_AND_CONFIG_MAP:** Supports both modern OIDC and legacy ConfigMap auth
- **No creator admin:** Forces explicit permission management

### **3. Advanced EKS Add-ons Configuration:**
```hcl
cluster_addons = {
  coredns    = {}                    # DNS resolution for cluster
  kube-proxy = {}                    # Network proxy on each node
  
  vpc-cni = {
    before_compute = true            # Deploy CNI before any nodes
    most_recent    = true           # Always use latest CNI version
    configuration_values = jsonencode({
      env = {
        # CRITICAL: Custom networking configuration
        AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG = "true"
        ENI_CONFIG_LABEL_DEF               = "topology.kubernetes.io/zone"
        
        # CRITICAL: Prefix delegation for high pod density
        ENABLE_PREFIX_DELEGATION = "true"
        WARM_PREFIX_TARGET       = "1"
      }
    })
  }
  
  eks-pod-identity-agent = {}        # Modern pod identity (replaces IRSA)
  aws-ebs-csi-driver = {
    most_recent              = true
    service_account_role_arn = module.ebs_csi_irsa_role.iam_role_arn
  }
}
```

**Add-on Deep Dive:**

#### **VPC-CNI Configuration (The Heart of Networking):**
```yaml
AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG: "true"
Purpose: Separates pod networking from node networking
Result: Pods get IPs from 100.96.32.0/21, nodes from 10.142.36.0/24

ENI_CONFIG_LABEL_DEF: "topology.kubernetes.io/zone"  
Purpose: Creates per-AZ networking configuration
Result: Pods in us-west-2a get IPs from one subnet, us-west-2b from another

ENABLE_PREFIX_DELEGATION: "true"
Purpose: Each ENI gets a /28 prefix (16 IPs) instead of single IPs
Result: ~110 pods per node vs ~17 with traditional CNI

WARM_PREFIX_TARGET: "1" 
Purpose: Keep one warm prefix ready for fast pod scheduling
Result: Faster pod startup times
```

### **4. Networking Configuration:**
```hcl
vpc_id                   = var.vpc_id              # vpc-078e600d57da47c6c
subnet_ids               = var.k8s_subnets         # Control plane subnets
control_plane_subnet_ids = var.k8s_subnets         # Same subnets for control plane

# The actual values from qas/variables.tfvars:
k8s_subnets = [
  "subnet-06582423b4bddaaef",  # leadertools.qas.k8s.prv.az1 (us-west-2a)
  "subnet-090a8b74a06b832c8"   # leadertools.qas.k8s.prv.az2 (us-west-2b)
]
```

**Network Design Decision:**
- **Control plane** runs in Kubernetes-specific private subnets
- **Worker nodes** also run in same subnets (collocated for performance)
- **Pods** get IPs from separate pod subnets via CNI custom networking

### **5. Node Groups Configuration:**
```hcl
eks_managed_node_groups = var.node_groups

# From qas/variables.tfvars:
node_groups = {
  eks_leadertools = {
    instance_types             = ["t3a.medium"]      # Cost-optimized instances
    min_size                   = 2                   # High availability minimum
    max_size                   = 3                   # Controlled scaling
    desired_size               = 2                   # Starting capacity
    disk_size                  = 20                  # Root volume size
    use_custom_launch_template = true               # Custom networking setup
    
    taints = [{
      key    = "CriticalAddonsOnly"                 # System workloads only
      value  = "true"
      effect = "NO_SCHEDULE"                        # App workloads use Karpenter
    }]
  }
}
```

**Node Group Strategy:**
- **Managed node group:** For critical system components (CoreDNS, ALB Controller)
- **Tainted nodes:** Prevent application workloads (those use Karpenter)
- **Small, stable:** 2-3 nodes for predictable system components
- **t3a.medium:** 2 vCPU, 4 GB RAM - sufficient for system workloads

### **6. Enhanced Security Group Rules:**
```hcl
node_security_group_additional_rules = {
  ingress_self_all = {
    description = "Node to node all ports/protocols"
    protocol    = "-1"            # All protocols
    from_port   = 0               # All ports
    to_port     = 0               # All ports
    type        = "ingress"       # Inbound rule
    self        = true            # From same security group
  }
}
```

**Why This Rule?**
- **Pod-to-pod communication:** Pods on different nodes need to communicate
- **Self-referencing:** Only allows traffic between nodes in same cluster
- **All protocols:** Supports TCP, UDP, ICMP for comprehensive connectivity
- **Security:** Still isolated from external networks

---

## 🌐 CNI Custom Networking Deep Dive

### **ENI Configuration per Availability Zone:**
```hcl
# Dynamic creation of ENIConfig for each AZ
resource "kubectl_manifest" "eni_config" {
  for_each = zipmap(
    [for subnet in data.aws_subnet.pod_azs : subnet.availability_zone],
    [for subnet in data.aws_subnet.pod_azs : subnet.id]
  )

  yaml_body = yamlencode({
    apiVersion = "crd.k8s.amazonaws.com/v1alpha1"
    kind       = "ENIConfig"
    metadata = {
      name = each.key                    # AZ name (us-west-2a, us-west-2b)
    }
    spec = {
      securityGroups = [
        module.eks.node_security_group_id,
      ]
      subnet = each.value               # Pod subnet ID for this AZ
    }
  })
}
```

**How This Works:**
1. **Data source** queries pod subnets to get AZ mappings
2. **zipmap** creates AZ → subnet ID mapping
3. **for_each** creates one ENIConfig per AZ
4. **kubectl_manifest** creates Kubernetes resources via Terraform

**Resulting ENIConfig Resources:**
```yaml
# us-west-2a ENIConfig
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
  name: us-west-2a
spec:
  securityGroups: ["sg-07019a3af3f18cfa1"]
  subnet: subnet-09d92cbf937adefe5     # Pod subnet AZ-A

# us-west-2b ENIConfig  
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
  name: us-west-2b
spec:
  securityGroups: ["sg-07019a3af3f18cfa1"] 
  subnet: subnet-0282b6517b060ced4     # Pod subnet AZ-B
```

### **Data Sources for Dynamic Configuration:**
```hcl
data "aws_subnet" "pod_azs" {
  for_each = toset(var.pod_subnets)
  id       = each.key
}

# Queries these subnets:
pod_subnets = [
  "subnet-09d92cbf937adefe5",  # 100.96.32.0/22 (AZ-A)
  "subnet-0282b6517b060ced4"   # 100.96.36.0/22 (AZ-B)
]
```

---

## 🔐 AWS Load Balancer Controller (`eks-alb-ingress.tf`)

### **1. IAM Role for Service Accounts (IRSA):**
```hcl
module "lb_role" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  version = "~> v5.58.0"

  role_name                              = "${var.environment}_${var.cluster_name}_eks_lb"
  attach_load_balancer_controller_policy = true    # AWS managed policy

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:aws-load-balancer-controller"]
    }
  }
}
```

**IRSA Flow:**
1. **EKS creates OIDC provider** for cluster
2. **IAM role trusts OIDC provider** with specific conditions
3. **Service account annotated** with IAM role ARN
4. **Pod gets JWT token** from Kubernetes
5. **AWS STS exchanges JWT** for temporary credentials

### **2. Service Account Creation:**
```hcl
resource "kubernetes_service_account" "service-account" {
  metadata {
    name      = "aws-load-balancer-controller"
    namespace = "kube-system"
    
    annotations = {
      "eks.amazonaws.com/role-arn"               = module.lb_role.iam_role_arn
      "eks.amazonaws.com/sts-regional-endpoints" = "true"    # Use regional STS
    }
  }
}
```

### **3. Helm Chart Deployment:**
```hcl
resource "helm_release" "lb" {
  name       = "aws-load-balancer-controller"
  chart      = "aws-load-balancer-controller"
  repository = "https://aws.github.io/eks-charts"         # Official AWS charts
  namespace  = "kube-system"
  version    = var.aws_alb_ingress_helm_version           # "1.5.0"

  values = [
    templatefile("${path.module}/values/alb-ingress/values.yaml", {
      region       = var.region          # us-west-2
      vpc_id       = var.vpc_id          # vpc-078e600d57da47c6c
      cluster_name = var.cluster_name    # leadertools-qas
    })
  ]
}
```

**What This Creates:**
- **Deployment:** ALB Controller pods in kube-system
- **CRDs:** Ingress class and target group binding resources
- **Webhooks:** Admission controllers for ingress validation
- **Service:** Internal service for webhook endpoints

---

## 🚀 Karpenter Auto-Scaling (`eks-karpenter.tf`)

### **1. Karpenter Module Configuration:**
```hcl
module "karpenter" {
  source  = "terraform-aws-modules/eks/aws//modules/karpenter"
  version = "~> 20.20"

  cluster_name = module.eks.cluster_name

  # Modern pod identity (replaces IRSA)
  enable_pod_identity             = true
  create_pod_identity_association = true

  # Additional IAM policies for node management
  node_iam_role_additional_policies = {
    AmazonSSMManagedInstanceCore = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
  }
}
```

### **2. Karpenter Helm Chart:**
```hcl
resource "helm_release" "karpenter" {
  name       = "karpenter"
  chart      = "karpenter"
  repository = "oci://public.ecr.aws/karpenter"    # OCI registry
  namespace  = "kube-system"
  version    = var.karpenter_helm_version          # "1.5.0"

  values = [
    <<-EOT
    serviceAccount:
      name: ${module.karpenter.service_account}
    settings:
      clusterName: ${module.eks.cluster_name}
      clusterEndpoint: ${module.eks.cluster_endpoint}
      interruptionQueue: ${module.karpenter.queue_name}
    EOT
  ]
}
```

### **3. EC2NodeClass (Karpenter v1 API):**
```hcl
resource "kubectl_manifest" "karpenter_node_class_v1" {
  yaml_body = yamlencode({
    apiVersion = "karpenter.k8s.aws/v1"
    kind       = "EC2NodeClass"
    metadata = {
      name = "default-v1"
    }
    spec = {
      amiSelectorTerms = [{
        alias = var.karpenter_ami_selector_terms    # "al2023@v20250620"
      }]
      role = module.karpenter.node_iam_role_name
      
      # Use same subnets as managed node groups
      subnetSelectorTerms = [
        for subnet_id in var.k8s_subnets : {
          id = subnet_id
        }
      ]
      
      securityGroupSelectorTerms = [{
        id = module.eks.node_security_group_id
      }]
      
      tags = merge({
        "karpenter.sh/discovery" = module.eks.cluster_name
        Name                     = "${var.cluster_name}-node"
      }, var.additional_tags)
    }
  })
}
```

### **4. NodePool Configuration:**
```hcl
resource "kubectl_manifest" "karpenter_node_pool_v1" {
  yaml_body = <<-YAML
    apiVersion: karpenter.sh/v1
    kind: NodePool
    metadata:
      name: default
    spec:
      template:
        spec:
          requirements:
            - key: kubernetes.io/arch
              operator: In
              values: ["amd64"]
            - key: kubernetes.io/os  
              operator: In
              values: ["linux"]
            - key: karpenter.sh/capacity-type
              operator: In
              values: ["on-demand", "spot"]        # Mixed capacity for cost savings
            - key: "karpenter.k8s.aws/instance-category"
              operator: In  
              values: ["c", "m", "r"]              # Compute, memory, general purpose
            - key: "karpenter.k8s.aws/instance-cpu"
              operator: In
              values: ["2", "4", "8"]              # 2-8 vCPU range
            - key: "karpenter.k8s.aws/instance-generation"
              operator: Gt
              values: ["2"]                        # Gen 3+ instances only
          
          kubelet:
            containerRuntime: containerd           # Modern container runtime
            systemReserved:                        # Reserve resources for system
              cpu: 100m
              memory: 100Mi
              
          nodeClassRef:
            group: karpenter.k8s.aws
            kind: EC2NodeClass
            name: default-v1
            
      limits:
        cpu: 1000                                  # Max 1000 vCPU across cluster
        
      disruption:
        consolidationPolicy: WhenEmpty             # Consolidate empty nodes
        consolidateAfter: 30s                     # Fast consolidation
  YAML
}
```

**Karpenter Strategy Explained:**
- **Mixed capacity:** Spot + On-demand for cost optimization (60% savings)
- **Instance flexibility:** Multiple categories and sizes for better availability
- **Modern instances:** Generation 3+ for better performance/cost ratio
- **Fast scaling:** 30-second consolidation for cost efficiency
- **Resource limits:** Prevent runaway scaling (max 1000 vCPU)

---

## 📊 Version Strategy & Compatibility

### **Terraform Versions (`versions.tf`):**
```hcl
terraform {
  required_version = ">= 1.5"                    # Modern Terraform features

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.40"                        # Latest AWS features
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"  
      version = ">= 2.20"                        # Kubernetes 1.33 support
    }
    helm = {
      source  = "hashicorp/helm"
      version = ">= 2.9"                         # OCI repository support
    }
    kubectl = {
      source  = "gavinbunney/kubectl"
      version = ">= 1.14"                        # Manifest management
    }
  }
}
```

### **Module Versions Used:**
```yaml
EKS Module: terraform-aws-modules/eks/aws ~> 20.20
  - Latest stable version with Kubernetes 1.33 support
  - Modern authentication modes (API_AND_CONFIG_MAP)
  - Pod identity agent support

IAM Module: terraform-aws-modules/iam/aws ~> 5.58.0  
  - IRSA support with OIDC providers
  - Latest AWS IAM features

Karpenter: v1.5.0
  - Latest Karpenter with v1 APIs
  - Improved performance and reliability
  - Better spot instance handling

ALB Controller: v1.5.0
  - Latest AWS Load Balancer Controller
  - Advanced ingress features
  - WebSocket and gRPC support
```

### **Kubernetes Version Strategy:**
```yaml
Cluster Version: "1.33" (Latest)
Benefits:
  ✅ Latest security patches
  ✅ Performance improvements  
  ✅ New API features
  ✅ Better resource management
  ✅ Enhanced networking capabilities

Add-on Compatibility:
  - vpc-cni: most_recent = true (auto-updates)
  - coredns: Latest compatible version
  - kube-proxy: Latest compatible version
  - ebs-csi-driver: most_recent = true
```

---

## 🔗 VPC & Aviatrix Integration

### **How Terraform Relates to Aviatrix:**

#### **1. VPC Configuration (Terraform-managed):**
```hcl
# Terraform creates/manages:
vpc_id = "vpc-078e600d57da47c6c"           # VPC boundary
k8s_subnets = [                           # EKS control plane/nodes
  "subnet-06582423b4bddaaef",             # Kubernetes AZ-A
  "subnet-090a8b74a06b832c8"              # Kubernetes AZ-B  
]
pod_subnets = [                           # Pod networking
  "subnet-09d92cbf937adefe5",             # Pod AZ-A (100.96.32.0/22)
  "subnet-0282b6517b060ced4"              # Pod AZ-B (100.96.36.0/22)
]
```

#### **2. Aviatrix Gateway Integration (External to Terraform):**
```yaml
Aviatrix Controller manages:
  - Gateway instances: i-0919b0c12ffebfc98, i-022003229928b80a1
  - Route table updates for corporate connectivity
  - Health checks and failover automation
  - VPN tunnel management

Terraform provisions the foundation:
  - VPC and subnets for gateway placement
  - Security groups allowing gateway traffic  
  - Route tables that Aviatrix can modify
  - IAM roles for Aviatrix controller access
```

#### **3. Route Table Integration Pattern:**
```hcl
# Terraform creates route tables with basic routes
resource "aws_route_table" "private" {
  vpc_id = var.vpc_id
  
  route {
    cidr_block = "10.142.32.0/21"
    gateway_id = "local"
  }
  
  route {
    cidr_block = "100.96.32.0/21"  
    gateway_id = "local"
  }
  
  # Aviatrix Controller adds corporate routes dynamically:
  # route {
  #   cidr_block = "208.75.9.40/32"
  #   instance_id = "i-0919b0c12ffebfc98"
  # }
}
```

### **4. Security Group Coordination:**
```hcl
# Terraform creates base security groups
resource "aws_security_group" "eks_nodes" {
  name_prefix = "${var.cluster_name}-node"
  vpc_id      = var.vpc_id

  # EKS-specific rules
  ingress {
    from_port = 443
    to_port   = 443  
    protocol  = "tcp"
    self      = true
  }
  
  # Aviatrix gateways can use these same security groups
  # or reference them in gateway configurations
}
```

---

## 🛠️ Terraform Advanced Concepts Used

### **1. Complex Variable Types:**
```hcl
variable "node_groups" {
  type = map(object({
    instance_types             = list(string)
    min_size                   = number
    max_size                   = number
    desired_size               = number
    disk_size                  = number
    use_custom_launch_template = bool
    taints = list(object({
      key    = string
      value  = string
      effect = string
    }))
  }))
  description = "Complex node group configuration"
}
```

### **2. Dynamic Resource Creation:**
```hcl
# Creates one ENIConfig per availability zone automatically
resource "kubectl_manifest" "eni_config" {
  for_each = zipmap(
    [for subnet in data.aws_subnet.pod_azs : subnet.availability_zone],
    [for subnet in data.aws_subnet.pod_azs : subnet.id]
  )
  # Dynamic manifest creation based on infrastructure discovery
}
```

### **3. Module Composition Pattern:**
```hcl
# Main EKS module
module "eks" { ... }

# Karpenter module depends on EKS
module "karpenter" {
  cluster_name = module.eks.cluster_name  # Reference EKS outputs
}

# IRSA roles depend on EKS OIDC provider
module "ebs_csi_irsa_role" {
  oidc_providers = {
    ex = {
      provider_arn = module.eks.oidc_provider_arn
    }
  }
}
```

### **4. Template Functions:**
```hcl
values = [
  templatefile("${path.module}/values/alb-ingress/values.yaml", {
    region       = var.region
    vpc_id       = var.vpc_id
    cluster_name = var.cluster_name
  })
]
```

### **5. Resource Dependencies:**
```hcl
depends_on = [
  module.eks.cluster,                    # Wait for cluster creation
  aws_eks_access_policy_association.sso, # Wait for access policies
  kubernetes_service_account.service-account, # Wait for service account
]
```

---

## 🎯 Interview Deep-Dive Questions

### **Q: "Walk me through the Terraform code for EKS networking"**
```
Answer: "Our Terraform uses a sophisticated approach:

1. CORE MODULE: We use the community terraform-aws-modules/eks/aws 
   module v20.20 which provides best practices and latest features

2. CNI CUSTOM NETWORKING: The vpc-cni addon is configured with 
   AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true and prefix delegation 
   to separate pod IPs from node IPs

3. DYNAMIC CONFIGURATION: We use for_each loops with data sources 
   to create ENIConfig resources for each AZ automatically

4. MODULE COMPOSITION: Separate modules for EKS, Karpenter, and 
   IAM roles that reference each other's outputs for dependencies"
```

### **Q: "How does Terraform integrate with Aviatrix?"**
```
Answer: "Terraform and Aviatrix have complementary roles:

1. TERRAFORM FOUNDATION: Creates VPC, subnets, security groups, 
   and route tables that Aviatrix can modify

2. AVIATRIX AUTOMATION: The Aviatrix Controller uses AWS APIs 
   to dynamically update route tables with corporate network routes

3. IAM INTEGRATION: Terraform creates IAM roles that allow 
   Aviatrix Controller to manage AWS networking resources

4. INFRASTRUCTURE SEPARATION: Terraform manages the base 
   infrastructure, Aviatrix manages the dynamic routing policies"
```

### **Q: "Explain the version strategy for all components"**
```
Answer: "We use a modern, compatible version strategy:

1. TERRAFORM: >= 1.5 for latest features and performance
2. KUBERNETES: 1.33 (latest) for security and functionality  
3. EKS MODULE: ~> 20.20 for Kubernetes 1.33 support
4. KARPENTER: v1.5.0 with new v1 APIs for better performance
5. ADD-ONS: most_recent = true for automatic security updates
6. AMI: al2023@v20250620 (Amazon Linux 2023) for modern base image

This ensures we get latest features while maintaining compatibility."
```

### **Q: "How do you handle the complexity of CNI custom networking in Terraform?"**
```
Answer: "We use several advanced Terraform patterns:

1. DATA SOURCES: Query pod subnets to discover AZ mappings dynamically
2. FOR_EACH: Create ENIConfig resources for each AZ without hardcoding
3. ZIPMAP: Create AZ-to-subnet mappings for clean iteration
4. JSONENCODE: Configure complex CNI environment variables
5. KUBECTL PROVIDER: Manage Kubernetes resources alongside AWS resources

This allows the infrastructure to adapt automatically to subnet changes."
```

---

## 🏆 Terraform Best Practices Demonstrated

### **1. Module Versioning:**
```hcl
source  = "terraform-aws-modules/eks/aws"
version = "~> 20.20"    # Pin major.minor, allow patch updates
```

### **2. Variable Validation:**
```hcl
variable "vpc_id" {
  validation {
    error_message = "Must be valid AWS VPC ID, like: 'vpc-XXXXXXXXXXXXX'"
    condition     = can(regex("^vpc-[\\d\\w\\-]{8,20}$", var.vpc_id))
  }
}
```

### **3. Resource Tagging:**
```hcl
tags = merge({
  "karpenter.sh/discovery" = var.cluster_name
  Name                     = "${var.cluster_name}-node"
}, var.additional_tags)
```

### **4. Environment Separation:**
```hcl
# environments/qas/variables.tfvars
cluster_name = "leadertools-qas"
environment  = "qas"

# Different values for prod, dev, etc.
```

### **5. Output Values:**
```hcl
output "cluster_endpoint" {
  value = module.eks.cluster_endpoint
}
output "cluster_security_group_id" {
  value = module.eks.cluster_security_group_id
}
```

---

## 🚀 Key Takeaways

This Terraform implementation demonstrates **enterprise-level infrastructure as code** with:

✅ **Advanced networking** - CNI custom networking with prefix delegation  
✅ **Modern authentication** - OIDC + IRSA for zero-trust security  
✅ **Auto-scaling** - Karpenter with intelligent instance selection  
✅ **High availability** - Multi-AZ with proper dependencies  
✅ **Cost optimization** - Spot instances and right-sizing  
✅ **Operational excellence** - Automated deployment and management  
✅ **Security best practices** - Least privilege and network isolation  

The code showcases sophisticated Terraform patterns that create a production-ready, enterprise-grade EKS environment! 🏗️