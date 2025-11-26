# Repository Overview

## What is this repository?

This is a **Terraform module** that deploys the [AWS Load Balancer Controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller) into an existing Amazon EKS (Elastic Kubernetes Service) cluster.

## Purpose

The AWS Load Balancer Controller manages AWS Elastic Load Balancers for a Kubernetes cluster. It provisions:
- **Application Load Balancers (ALB)** for Kubernetes Ingress resources
- **Network Load Balancers (NLB)** for Kubernetes Service resources

This module automates the deployment and IAM configuration required to run the controller.

## Key Features

✅ **Simple Deployment** - Deploys AWS Load Balancer Controller via Helm chart  
✅ **IAM Integration** - Automatically creates and configures IAM roles with OIDC authentication  
✅ **Flexible Configuration** - Supports custom namespaces, service accounts, and Helm values  
✅ **RBAC Support** - Optional cross-namespace secret access for the controller  
✅ **GovCloud Compatible** - Works with AWS GovCloud partitions  
✅ **Production Ready** - Includes permissions boundaries and tagging support

## Quick Start

```hcl
module "load_balancer_controller" {
  source = "git::https://github.com/DNXLabs/terraform-aws-eks-lb-controller.git"

  cluster_identity_oidc_issuer     = module.eks_cluster.cluster_oidc_issuer_url
  cluster_identity_oidc_issuer_arn = module.eks_cluster.oidc_provider_arn
  cluster_name                     = module.eks_cluster.cluster_id
}
```

## Repository Structure

```
.
├── _variables.tf          # Input variables (cluster config, Helm settings, IAM options)
├── versions.tf            # Terraform and provider version constraints
├── iam.tf                 # IAM roles and policies for the controller (415 lines)
├── helm.tf                # Helm release configuration
├── namespace.tf           # Kubernetes namespace creation (optional)
├── role.tf                # RBAC roles for cross-namespace access (optional)
├── role.tpl.yaml          # Kubernetes Role template
├── role_binding.tpl.yaml  # Kubernetes RoleBinding template
├── README.md              # Module documentation with all variables
├── examples/
│   └── basic/             # Complete example with VPC and EKS cluster
└── .github/
    └── workflows/         # CI/CD for linting
```

## What It Creates

When you apply this module, it creates:

1. **IAM Role** - For the AWS Load Balancer Controller with OIDC trust relationship
2. **IAM Policy** - With permissions to manage ELBs, Target Groups, Security Groups, etc.
3. **Kubernetes Namespace** - (Optional) If not using kube-system
4. **Helm Release** - Deploys the aws-load-balancer-controller chart
5. **Kubernetes Service Account** - With IAM role annotation for IRSA (IAM Roles for Service Accounts)
6. **RBAC Roles** - (Optional) For accessing secrets in other namespaces

## Prerequisites

Before using this module, you need:

- ✅ An existing **EKS cluster** (this module doesn't create the cluster)
- ✅ **OIDC provider** configured for the EKS cluster (for IAM role authentication)
- ✅ **Helm** provider configured in your Terraform
- ✅ **kubectl** provider configured

## Configuration Options

### Required Variables
- `cluster_name` - Your EKS cluster name
- `cluster_identity_oidc_issuer` - OIDC issuer URL from your EKS cluster
- `cluster_identity_oidc_issuer_arn` - OIDC provider ARN

### Key Optional Variables
- `enabled` - Enable/disable the module (default: true)
- `namespace` - Kubernetes namespace (default: kube-system)
- `helm_chart_version` - Controller version (default: 1.10.1)
- `settings` - Custom Helm chart values
- `permissions_boundary` - IAM permissions boundary ARN
- `tags` - Tags for IAM resources
- `roles` - RBAC roles for cross-namespace secret access

## Use Cases

This module is perfect for:

1. **Application Ingress** - Automatically provision ALBs for your Kubernetes Ingress resources
2. **Service Load Balancing** - Create NLBs for LoadBalancer-type Services
3. **Multi-Tenant Clusters** - Use RBAC roles to manage secrets across namespaces
4. **Regulated Environments** - Supports permissions boundaries and custom ARN formats for GovCloud

## Example Deployment Flow

1. **You apply this module** → Creates IAM role and deploys Helm chart
2. **Controller starts in your cluster** → Watches for Ingress/Service resources
3. **You create a Kubernetes Ingress** → Controller provisions an ALB automatically
4. **You create a LoadBalancer Service** → Controller provisions an NLB automatically

## Dependencies

This module uses:
- **Terraform** >= 0.13 (recommend upgrading to >= 1.0)
- **AWS Provider** >= 3.35
- **Helm Provider** >= 1.0, < 3.0
- **Kubernetes Provider** >= 1.10.0, < 3.0.0
- **kubectl Provider** >= 1.9.4

## Related Resources

- [AWS Load Balancer Controller Documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [AWS Load Balancer Controller GitHub](https://github.com/kubernetes-sigs/aws-load-balancer-controller)
- [EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/)

## Health Status

📊 **Overall Module Health: 72/100**

A comprehensive health assessment is available in [MODULE_HEALTH_REPORT.md](MODULE_HEALTH_REPORT.md), which includes:
- Structure analysis (70/100)
- Syntax & style validation (85/100)
- Currency & deprecation check (62/100)
- Prioritized recommendations for improvements

### Quick Wins for Improvement
1. Update Terraform version constraint to >= 1.0.0
2. Add outputs.tf for better module composability
3. Update to latest Helm chart version

## Contributing

- **Maintainer**: [DNX Solutions](https://github.com/DNXLabs)
- **License**: Apache 2.0
- **CI/CD**: GitHub Actions for linting

See [CONTRIBUTING.md](CONTRIBUTING.md) for contribution guidelines.

## Support

- 🐛 **Issues**: [GitHub Issues](https://github.com/DNXLabs/terraform-aws-eks-lb-controller/issues)
- 📖 **Documentation**: See [README.md](README.md) for detailed variable documentation

---

**TL;DR**: This Terraform module makes it easy to deploy the AWS Load Balancer Controller to your EKS cluster, handling all the IAM setup automatically. Just point it at your existing EKS cluster and it takes care of the rest.
