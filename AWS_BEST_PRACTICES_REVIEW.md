# AWS Best Practices Review

**Repository:** terraform-aws-eks-lb-controller  
**Review Date:** 2025-11-26  
**Reviewer:** AWS Terraform Specialist

## Executive Summary

This report provides a comprehensive review of AWS resources used in the terraform-aws-eks-lb-controller module against current AWS best practices, security recommendations, and official documentation. The module deploys the AWS Load Balancer Controller to an EKS cluster using IAM roles for service accounts (IRSA).

## Resources Analyzed

### AWS Resources (count: 4)
- `aws_iam_policy` (count: 1)
- `aws_iam_role` (count: 1)
- `aws_iam_policy_document` (count: 2)
- `aws_iam_role_policy_attachment` (count: 1)

### Kubernetes Resources (count: 3)
- `helm_release` (count: 1)
- `kubernetes_namespace` (count: 1)
- `kubectl_manifest` (count: 2)

## Findings

### Resource: aws_iam_policy.lb_controller

**Current Configuration:**
- Creates IAM policy for AWS Load Balancer Controller with comprehensive permissions
- Permissions cover EC2, ELB/ALB/NLB, ACM, WAF/WAFv2, Shield, and Cognito services
- Includes 60+ IAM actions across multiple AWS services
- Uses resource wildcards (`"*"`) for many actions

**AWS Documentation:**
- [Security best practices in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AWS Load Balancer Controller IAM Policy](https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/main/docs/install/iam_policy.json)
- [Prepare for least-privilege permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/getting-started-reduce-permissions.html)

**Findings:**

#### 1. IAM Policy Alignment with Official AWS LBC Policy
**Priority: MEDIUM**

The module includes recently added ELB v2 API actions that align with the latest AWS Load Balancer Controller requirements:
- `elasticloadbalancing:DescribeTrustStores` (line 58)
- `elasticloadbalancing:DescribeListenerAttributes` (line 59)
- `elasticloadbalancing:DescribeCapacityReservation` (line 60)
- `elasticloadbalancing:ModifyListenerAttributes` (line 291)
- `elasticloadbalancing:ModifyCapacityReservation` (line 292)
- `elasticloadbalancing:ModifyIpPools` (line 293)

**Recommendation:** The IAM policy is well-maintained and includes the latest API actions. Continue to monitor the official AWS Load Balancer Controller GitHub repository for policy updates. The current default Helm chart version is `1.10.1` which may be outdated.

**Reference:** [AWS Load Balancer Controller v2.14.1 IAM Policy](https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json)

#### 2. Resource Wildcard Usage
**Priority: MEDIUM**

Multiple policy statements use `"resources": ["*"]` which is common for AWS Load Balancer Controller but could be more restrictive:
- Lines 14-28: IAM service-linked role creation
- Lines 62-66: EC2 and ELB describe operations
- Lines 88-92: ACM, WAF, Shield operations
- Lines 99-103: EC2 security group operations

**Recommendation:** While wildcards are standard for this controller due to dynamic resource creation, consider:
1. Using condition keys to scope permissions where possible (already implemented for cluster tags)
2. Document that these wildcards are necessary for the controller's dynamic resource management
3. Ensure the condition-based restrictions (e.g., `elbv2.k8s.aws/cluster` tag) are maintained

**AWS Best Practice:** "Grant only the permissions required for users, groups, and roles to perform their functions. Avoid using broad or wildcard permissions unless necessary for dynamic resource management."

**Reference:** [IAM Least Privilege Best Practices](https://moldstud.com/articles/p-a-comprehensive-guide-to-aws-iam-policy-best-practices-for-enhanced-security)

#### 3. IPAM and Route Table Permissions
**Priority: LOW**

The policy includes newer EC2 IPAM-related permissions:
- `ec2:GetSecurityGroupsForVpc` (line 45)
- `ec2:DescribeIpamPools` (line 46)
- `ec2:DescribeRouteTables` (line 47)

**Recommendation:** These permissions are appropriate for AWS Load Balancer Controller's advanced VPC integration features. No changes needed.

**Reference:** [EC2 API Actions](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonec2.html)

---

### Resource: aws_iam_role.lb_controller

**Current Configuration:**
- IAM role with OIDC federation for EKS service accounts (IRSA)
- Trust policy restricts assumption to specific service account in specific namespace
- Supports optional permissions boundary
- Uses `sts:AssumeRoleWithWebIdentity` action

**AWS Documentation:**
- [IAM roles for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [Create an IAM OIDC provider for your cluster](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)

**Findings:**

#### 1. IRSA Security Implementation
**Priority: HIGH (Security - Compliant)**

The module correctly implements IAM Roles for Service Accounts (IRSA) best practices:
- ✅ Uses OIDC federation instead of long-lived credentials
- ✅ Restricts trust to specific namespace and service account (lines 390-396)
- ✅ Uses condition-based trust policy with `StringEquals` test
- ✅ Supports optional permissions boundary for additional security layer

**Recommendation:** This is a security best practice. The implementation is correct and follows AWS recommendations.

**AWS Best Practice:** "Use IAM roles instead of long-lived access keys. IRSA provides temporary credentials that expire automatically (typically ≤1 hour), reducing risk if credentials leak."

**Reference:** [EKS Security Best Practices for IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)

#### 2. Role Naming Convention
**Priority: LOW**

The module uses a sensible default naming pattern: `${cluster_name}-alb-ingress` but allows override via `role_name` variable.

**Recommendation:** Consider documenting the naming convention in the README. Current implementation is flexible and appropriate.

---

### Resource: aws_iam_policy_document (Trust Policy)

**Current Configuration:**
- Trust policy document for OIDC federation
- Scoped to specific service account: `system:serviceaccount:${namespace}:${service_account_name}`
- Uses OIDC issuer validation

**Findings:**

#### 1. Trust Policy Security
**Priority: HIGH (Security - Compliant)**

The trust policy correctly implements security best practices:
- ✅ Restricts to federated OIDC provider
- ✅ Uses namespace and service account name in condition
- ✅ Validates OIDC subject claim
- ✅ Effect is explicitly set to "Allow" with proper scoping

**Recommendation:** No changes needed. This follows AWS security best practices for IRSA.

**Reference:** [IAM Roles for Service Accounts Security](https://iwconnect.com/elevating-eks-security-through-iam-roles/)

#### 2. OIDC Provider Validation
**Priority: MEDIUM**

The trust policy uses `StringEquals` for OIDC issuer validation (line 391-392), which is the recommended approach.

**Recommendation:** Ensure users provide the correct OIDC issuer URL without the `https://` prefix since the module strips it (line 392). This is correctly implemented.

---

### Resource: helm_release.lb_controller

**Current Configuration:**
- Helm chart: `aws-load-balancer-controller`
- Default version: `1.10.1`
- Repository: `https://aws.github.io/eks-charts`
- Creates service account with IAM role annotation

**AWS Documentation:**
- [Install AWS Load Balancer Controller with Helm](https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html)
- [AWS Load Balancer Controller GitHub](https://github.com/kubernetes-sigs/aws-load-balancer-controller)

**Findings:**

#### 1. Helm Chart Version
**Priority: HIGH**

The default Helm chart version is `1.10.1`, which may be outdated. The latest version as of November 2024 is approximately v2.14.x of the controller.

**Recommendation:** Update the default `helm_chart_version` variable to a more recent version. Check the [EKS Charts repository](https://github.com/aws/eks-charts/tree/master/stable/aws-load-balancer-controller) for the latest stable version.

**Note:** The version is a variable, so users can override it, but the default should represent a tested, recent stable version.

**Reference:** [AWS Load Balancer Controller Releases](https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases)

#### 2. Service Account Configuration
**Priority: MEDIUM (Best Practice - Compliant)**

The Helm release correctly:
- ✅ Enables RBAC creation (`rbac.create = true`)
- ✅ Creates service account (`serviceAccount.create = true`)
- ✅ Annotates service account with IAM role ARN
- ✅ Uses the standard EKS annotation format: `eks.amazonaws.com/role-arn`

**Recommendation:** This is correctly implemented according to AWS best practices. No changes needed.

---

### Security Groups and Tagging

**Current Configuration:**
The IAM policy includes extensive security group management permissions with tag-based conditions using `elbv2.k8s.aws/cluster` tags.

**Findings:**

#### 1. Tag-Based Resource Scoping
**Priority: HIGH (Security - Compliant)**

The module implements tag-based scoping for security operations:
- ✅ Requires cluster tags for security group creation (lines 124-143)
- ✅ Requires cluster tags for security group modifications (lines 186-196)
- ✅ Requires cluster tags for load balancer creation (lines 209-217)
- ✅ Prevents tag manipulation on untagged resources (lines 154-173)

**Recommendation:** This is an excellent security practice. The tag-based conditions limit the blast radius if credentials are compromised. Ensure users understand they need to properly tag their VPC subnets for the controller to function.

**AWS Best Practice:** "Use conditional policies and permission boundaries. Apply conditions (e.g., resource tags) in IAM policies to restrict when and how permissions can be used."

**Reference:** [AWS IAM Security Best Practices](https://spacelift.io/blog/aws-iam-best-practices)

#### 2. Subnet Tagging Requirements
**Priority: HIGH (Documentation)**

The Load Balancer Controller requires specific subnet tags that are not mentioned in the module documentation:
- For public subnets: `kubernetes.io/role/elb=1`
- For private subnets: `kubernetes.io/role/internal-elb=1`

**Recommendation:** Add subnet tagging requirements to the README.md documentation to help users avoid common deployment issues.

**Reference:** [EKS Load Balancer Controller Installation Guide](https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html)

---

### Provider Version Constraints

**Current Configuration (versions.tf):**
```hcl
terraform >= 0.13
aws >= 3.35
helm >= 1.0, < 3.0
kubernetes >= 1.10.0, < 3.0.0
kubectl >= 1.9.4
```

**Findings:**

#### 1. AWS Provider Version
**Priority: MEDIUM**

The minimum AWS provider version is `>= 3.35`. Current AWS provider is v5.x, which includes:
- Improved EKS and ELB resource support
- Better IAM policy validation
- Enhanced security features

**Recommendation:** Consider updating the minimum version constraint to `>= 4.0` or `>= 5.0` for better security and feature support, while testing for breaking changes.

**Reference:** [Terraform AWS Provider Changelog](https://github.com/hashicorp/terraform-provider-aws/blob/main/CHANGELOG.md)

#### 2. Terraform Version
**Priority: LOW**

Minimum Terraform version is `>= 0.13`. This is quite old (released August 2020).

**Recommendation:** Consider updating to `>= 1.0` for better stability and security features, as Terraform 1.x includes significant improvements.

---

### Additional Security Considerations

#### 1. WAF Regional Deprecation
**Priority: MEDIUM**

The IAM policy includes WAF Regional (`waf-regional`) permissions:
- `waf-regional:GetWebACL`
- `waf-regional:GetWebACLForResource`
- `waf-regional:AssociateWebACL`
- `waf-regional:DisassociateWebACL`

**AWS Status:** AWS WAF Classic (including waf-regional) is deprecated. AWS recommends migrating to WAFv2.

**Recommendation:** Document that waf-regional permissions are included for backward compatibility but WAFv2 is recommended for new deployments. The policy correctly includes both `waf-regional` and `wafv2` actions.

**Reference:** [AWS WAF Classic Migration](https://docs.aws.amazon.com/waf/latest/developerguide/waf-chapter.html)

#### 2. Shield Protection Permissions
**Priority: LOW**

The policy includes AWS Shield permissions:
- `shield:GetSubscriptionState`
- `shield:DescribeProtection`
- `shield:CreateProtection`
- `shield:DeleteProtection`

**Recommendation:** These are appropriate for production deployments where DDoS protection is required. Document that Shield Standard is free but Shield Advanced requires a subscription.

**Reference:** [AWS Shield Pricing](https://aws.amazon.com/shield/pricing/)

---

## Recommended Improvements

### Priority: HIGH

1. **Update Default Helm Chart Version**
   - Current: `1.10.1`
   - Recommended: Check latest stable version from [eks-charts](https://github.com/aws/eks-charts)
   - Impact: Security patches, bug fixes, new features

2. **Add Subnet Tagging Documentation**
   - Document required subnet tags in README.md
   - Impact: Prevents deployment issues, improves user experience

3. **Maintain IAM Policy Alignment**
   - Regularly compare with official AWS LBC policy
   - Set up automation to check for policy drift
   - Impact: Security, compatibility

### Priority: MEDIUM

1. **Update Provider Version Constraints**
   - AWS provider: `>= 4.0` or `>= 5.0`
   - Terraform: `>= 1.0`
   - Impact: Better security, features, and support

2. **Enhance Documentation**
   - Add AWS documentation links to README
   - Document WAF Classic deprecation
   - Explain tag-based security scoping
   - Impact: Better user understanding, fewer issues

3. **Variable Defaults**
   - Review `arn_format` default for GovCloud use cases
   - Consider making `tags` more prominent in examples
   - Impact: Improved tracking and compliance

### Priority: LOW

1. **Add Output Values**
   - Currently: "No output"
   - Recommended: Export IAM role ARN, role name
   - Impact: Easier integration with other modules

2. **Example Enhancements**
   - Add example with permissions boundary
   - Add example with custom IAM policy modifications
   - Impact: Better guidance for advanced use cases

---

## Deprecation Warnings

### No Critical Deprecations Found

After reviewing AWS API changes and documentation:

✅ **Elastic Load Balancing v2 API**: No deprecated actions found in 2024. AWS continues to add new features without removing existing APIs.

✅ **IAM APIs**: All IAM actions used in this module are current and supported.

⚠️ **WAF Classic (waf-regional)**: Deprecated by AWS, but policy includes both Classic and WAFv2 for compatibility. Recommend WAFv2 for new deployments.

**Reference:** 
- [AWS API Changes Archive](https://awsapichanges.info/archive/service/elasticloadbalancing/)
- [ELB v2 Document History](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/doc-history.html)

---

## Security Summary

### Strengths

1. ✅ **IRSA Implementation**: Correctly implements IAM Roles for Service Accounts with OIDC federation
2. ✅ **Tag-Based Scoping**: Uses resource tags to limit permissions scope
3. ✅ **Condition-Based Policies**: Implements IAM condition keys for security group and load balancer operations
4. ✅ **Temporary Credentials**: Uses STS AssumeRoleWithWebIdentity for short-lived credentials
5. ✅ **Least Privilege Attempt**: While using wildcards, scopes them with conditions where possible
6. ✅ **Permissions Boundary Support**: Allows optional permissions boundary for additional security layer

### Areas for Improvement

1. 📝 **Documentation**: Add subnet tagging requirements and AWS best practices references
2. 📝 **Version Updates**: Update default Helm chart version and provider constraints
3. 📝 **Policy Maintenance**: Establish process for regular comparison with official AWS LBC policy
4. 📝 **Examples**: Add more examples demonstrating security features (permissions boundary, custom roles)

---

## Compliance and Best Practices Alignment

| AWS Best Practice | Status | Notes |
|------------------|--------|-------|
| Least Privilege | ✅ Partial | Uses wildcards where necessary, scoped with conditions |
| Temporary Credentials | ✅ Yes | IRSA provides short-lived credentials |
| No Hardcoded Credentials | ✅ Yes | Uses OIDC federation |
| MFA Enforcement | N/A | Not applicable to service accounts |
| Regular Audits | 📝 Recommended | Document audit procedures |
| IAM Roles Over Users | ✅ Yes | Uses IAM roles exclusively |
| Permission Boundaries | ✅ Supported | Optional parameter available |
| Resource Tagging | ✅ Yes | Implements tag-based controls |
| Condition-Based Policies | ✅ Yes | Multiple condition blocks |
| Service-Specific Roles | ✅ Yes | Dedicated role for LB controller |

**Legend:**
- ✅ Implemented/Compliant
- ⚠️ Partially Implemented
- ❌ Not Implemented
- 📝 Recommended Enhancement
- N/A Not Applicable

---

## Conclusion

The **terraform-aws-eks-lb-controller** module demonstrates strong security practices and good alignment with AWS best practices for IAM and EKS. The implementation of IRSA with tag-based scoping is particularly commendable. The main recommendations focus on documentation improvements, version updates, and maintaining alignment with the official AWS Load Balancer Controller IAM policy as it evolves.

### Overall Rating: **GOOD** ⭐⭐⭐⭐

The module is production-ready with the following priorities:
1. Update documentation (especially subnet tagging requirements)
2. Review and update default Helm chart version
3. Establish IAM policy maintenance process
4. Consider updating provider version constraints

---

## References

### Official AWS Documentation
- [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [IAM Roles for Service Accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [Install AWS Load Balancer Controller with Helm](https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html)
- [ELB v2 Actions Reference](https://docs.aws.amazon.com/service-authorization/latest/reference/list_awselasticloadbalancingv2.html)
- [Prepare for Least-Privilege Permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/getting-started-reduce-permissions.html)

### AWS Load Balancer Controller
- [GitHub Repository](https://github.com/kubernetes-sigs/aws-load-balancer-controller)
- [Official IAM Policy](https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json)
- [Helm Chart Repository](https://github.com/aws/eks-charts/tree/master/stable/aws-load-balancer-controller)

### Security Best Practices
- [AWS IAM Policy Best Practices](https://moldstud.com/articles/p-a-comprehensive-guide-to-aws-iam-policy-best-practices-for-enhanced-security)
- [AWS IAM Security Best Practices - Spacelift](https://spacelift.io/blog/aws-iam-best-practices)
- [EKS Security Through IAM Roles](https://iwconnect.com/elevating-eks-security-through-iam-roles/)
- [IAM Least Privilege Guide](https://www.datadoghq.com/blog/iam-least-privilege/)

### API References
- [ELB Document History](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/doc-history.html)
- [AWS API Changes Archive](https://awsapichanges.info/archive/service/elasticloadbalancing/)
- [EC2 Service Authorization](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonec2.html)

---

**Report Generated:** 2025-11-26  
**Module Version:** Current main branch  
**Review Methodology:** Code analysis + AWS documentation verification + security best practices audit
