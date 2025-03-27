# IMDSv2 Enforcement with AWS CloudFormation Guard Hook

This repository contains AWS CloudFormation Guard rules and deployment templates to enforce IMDSv2 across your AWS organization. The implementation uses AWS CloudFormation Guard Hook, which is a managed hook service that doesn't require Lambda functions and is easier to maintain.

## Overview

This solution enforces IMDSv2 at several levels:

1. **Preventative Control**: Using CloudFormation Guard Hook to block deployments of non-compliant resources
2. **Enterprise-Wide Enforcement**: Deploying the hook across all accounts in your organization
3. **Control Tower Integration**: Working with your existing Control Tower and Landing Zone Accelerator setup

## What are Guard Hooks?

Guard Hooks are a fully managed CloudFormation hook type that uses the Guard domain-specific language (DSL) for policy enforcement. Unlike Lambda-backed hooks, Guard Hooks:

- Don't require managing Lambda functions
- Use a simple, declarative language for rules
- Are fully managed by AWS
- Can be easily tested and validated

## Repository Structure

```
imdsv2-guard-hook/
├── guard-rules/
│   ├── ec2-imdsv2.guard               # IMDSv2 enforcement rules
│   └── test/                          # Test cases for validation
├── deployment/
│   ├── guardhook-template.yaml        # CloudFormation template for hook deployment
│   └── org-deployment.yaml            # Organization-wide deployment template
└── iam/                               # IAM roles required for the hook
```

## Guard Rules Explained

Our Guard rule (`ec2-imdsv2.guard`) enforces that:

1. All EC2 instances have `MetadataOptions.HttpTokens` set to `"required"`
2. All Launch Templates have `LaunchTemplateData.MetadataOptions.HttpTokens` set to `"required"`
3. All Launch Configurations have `MetadataOptions.HttpTokens` set to `"required"`

## Implementation Steps

### Prerequisites

- An S3 bucket to store Guard rules
- Admin access to your AWS Organizations management account
- Control Tower with Landing Zone Accelerator (optional)

### Step 1: Upload Guard Rules to S3

1. Package the Guard rules:
   ```bash
   zip -r imdsv2-rules.zip guard-rules/ec2-imdsv2.guard
   ```

2. Upload to your S3 bucket:
   ```bash
   aws s3 cp imdsv2-rules.zip s3://your-organization-bucket/guardhooks/
   ```

### Step 2: Deploy the Guard Hook in Management Account

1. Deploy the `guardhook-template.yaml` CloudFormation template with the following parameters:
   - **GuardRulesBucket**: Your S3 bucket name
   - **GuardRulesKey**: `guardhooks/imdsv2-rules.zip`
   - **HookName**: `Security::EC2::IMDSv2Enforcement` (or your preferred name)
   - **FailureMode**: `FAIL`
   - **LogBucket**: (Optional) A bucket to store detailed evaluation logs

   ```bash
   aws cloudformation create-stack \
     --stack-name imdsv2-guard-hook \
     --template-body file://deployment/guardhook-template.yaml \
     --parameters \
       ParameterKey=GuardRulesBucket,ParameterValue=your-organization-bucket \
       ParameterKey=GuardRulesKey,ParameterValue=guardhooks/imdsv2-rules.zip \
     --capabilities CAPABILITY_IAM
   ```

### Step 3: Deploy Hook Across Your Organization

1. Identify your Organizational Unit IDs

2. Deploy the `org-deployment.yaml` CloudFormation template:
   ```bash
   aws cloudformation create-stack \
     --stack-name imdsv2-hook-org-deployment \
     --template-body file://deployment/org-deployment.yaml \
     --parameters \
       ParameterKey=HookName,ParameterValue=Security::EC2::IMDSv2Enforcement \
       ParameterKey=TargetOUs,ParameterValue=\"ou-abcd-1example,ou-efgh-2example\" \
     --capabilities CAPABILITY_IAM
   ```

### Step 4: Integration with Landing Zone Accelerator (Optional)

Add the following to your LZA customizations:

```json
// customizations-config.json
"cloudFormationStackSets": [
  {
    "name": "imdsv2-guard-hook-configuration",
    "description": "Deploys IMDSv2 enforcement hook to all accounts",
    "template": "imdsv2-hook-org-deployment.yaml",
    "regions": ["us-east-1"],
    "deploymentTargets": {
      "organizationalUnits": ["Security", "Infrastructure", "Workloads"]
    }
  }
]
```

## Testing the Guard Hook

You can test the Guard rules locally before deployment:

```bash
# Install CloudFormation Guard
pip install cloudformation-guard

# Test with the passing case
cfn-guard validate -r guard-rules/ec2-imdsv2.guard -d guard-rules/test/ec2-imdsv2-pass.json

# Test with the failing case
cfn-guard validate -r guard-rules/ec2-imdsv2.guard -d guard-rules/test/ec2-imdsv2-fail.json
```

## Comprehensive Enforcement Strategy

For a complete IMDSv2 enforcement strategy:

1. **Guard Hook (this repository)**: Preventive control that blocks non-compliant resources
2. **AWS Config Rules**: Detective control to find existing non-compliant resources
3. **SSM Automation**: Remediation for existing non-compliant resources
4. **SCPs**: Organization-level policy enforcement

## Verification

After deployment, attempt to create an EC2 instance or Launch Template without IMDSv2. The operation should fail with a clear error message explaining the requirements.

## Troubleshooting

If you encounter issues:

1. Check the CloudWatch logs for the hook
2. If you configured a log bucket, review the detailed Guard evaluation logs
3. Test your Guard rules locally using the cfn-guard CLI
