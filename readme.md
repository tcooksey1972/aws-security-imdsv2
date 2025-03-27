# IMDSv2 Enforcement with AWS CloudFormation Guard Hook

This repository contains AWS CloudFormation Guard rules and deployment templates to enforce IMDSv2 across your AWS organization. The implementation uses AWS CloudFormation Guard Hook, which is a managed hook service that doesn't require Lambda functions and is easier to maintain.

## Overview

This solution enforces IMDSv2 at several levels:

1. **Preventative Control**: Using CloudFormation Guard Hook to block deployments of non-compliant resources
2. **Enterprise-Wide Enforcement**: Deploying the hook across all accounts in your organization
3. **Control Tower Integration**: Working with your existing Control Tower and Landing Zone Accelerator setup

## Solution Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS Organization                          │
│                                                                  │
│  ┌─────────────────────┐        ┌───────────────────────────┐   │
│  │   Management Acct   │        │     Member Accounts        │   │
│  │                     │        │                           │   │
│  │  ┌───────────────┐  │Deploy  │  ┌────────────────────┐   │   │
│  │  │ S3 Bucket     │  │via     │  │                    │   │   │
│  │  │ ┌───────────┐ │  │Stack   │  │  CloudFormation    │   │   │
│  │  │ │IMDSv2 Rule│ │  │Set     │  │  Guard Hook        │◄──┼───┼──┐
│  │  │ └───────────┘ │  │        │  │                    │   │   │  │
│  │  └───────┬───────┘  │        │  └────────────────────┘   │   │  │
│  │          │          │        │            ▲               │   │  │
│  │          ▼          │        │            │               │   │  │
│  │  ┌───────────────┐  │        │            │ Evaluate      │   │  │
│  │  │               │  │        │            │               │   │  │
│  │  │ Guard Hook    │  │        │  ┌─────────┴────────────┐  │   │  │
│  │  │ Definition    │  │        │  │                      │  │   │  │
│  │  │               │──┼────────┼─►│  CloudFormation      │  │   │  │
│  │  └───────────────┘  │        │  │  Deployments         │  │   │  │
│  │                     │        │  │                      │  │   │  │
│  │  ┌───────────────┐  │        │  └──────────────────────┘  │   │  │
│  │  │               │  │        │                            │   │  │
│  │  │ StackSet for  │  │        └────────────────────────────┘   │  │
│  │  │ Org-Wide      │──┼─────────────────────────────────────────┘  │
│  │  │ Deployment    │  │                                            │
│  │  │               │  │                                            │
│  │  └───────────────┘  │                                            │
│  └─────────────────────┘                                            │
└─────────────────────────────────────────────────────────────────────┘
```

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

## CI/CD Implementation

This repository includes a GitHub Actions workflow for automating the deployment pipeline. The workflow:

1. Packages the Guard rules
2. Deploys to a development environment first with WARN mode
3. Promotes to production with FAIL mode after testing
4. Optionally deploys organization-wide

The CI/CD workflow provides:
- Automatic testing of Guard rules before deployment
- Progressive deployment with safeguards
- Consistent versioning with Git SHA for tracking

## Operational Considerations

### Monitoring

1. **CloudWatch Logs**: Monitor the CloudWatch log groups for the Guard Hook to track evaluations and potential failures
   ```bash
   # Set up a CloudWatch alarm for hook failures
   aws cloudwatch put-metric-alarm \
     --alarm-name IMDSv2GuardHookFailures \
     --metric-name HookFailedEvaluations \
     --namespace AWS/CloudFormation \
     --statistic Sum \
     --period 300 \
     --threshold 1 \
     --comparison-operator GreaterThanOrEqualToThreshold \
     --dimensions Name=HookName,Value=Security::EC2::IMDSv2Enforcement \
     --evaluation-periods 1 \
     --alarm-actions <your-sns-topic-arn>
   ```

2. **S3 Access Logs**: Enable access logging on your S3 bucket containing the Guard rules to monitor for unauthorized access attempts

### Maintenance Procedures

#### Updating Rules

1. Modify the `ec2-imdsv2.guard` file with your changes
2. Test locally using the test cases
3. Deploy through your CI/CD pipeline or manually:
   ```bash
   # Package and upload the updated rules
   zip -r imdsv2-rules-v2.zip guard-rules/ec2-imdsv2.guard
   aws s3 cp imdsv2-rules-v2.zip s3://your-organization-bucket/guardhooks/
   
   # Update the CloudFormation stack with the new key
   aws cloudformation update-stack \
     --stack-name imdsv2-guard-hook \
     --parameters \
       ParameterKey=GuardRulesKey,ParameterValue=guardhooks/imdsv2-rules-v2.zip \
       ParameterKey=GuardRulesBucket,UsePreviousValue=true \
       ParameterKey=HookName,UsePreviousValue=true \
       ParameterKey=FailureMode,UsePreviousValue=true \
     --use-previous-template
   ```

#### Rollback Procedures

If issues are encountered after deployment:

1. **For rule issues**: Roll back to the previous version of the rules:
   ```bash
   aws cloudformation update-stack \
     --stack-name imdsv2-guard-hook \
     --parameters \
       ParameterKey=GuardRulesKey,ParameterValue=guardhooks/imdsv2-rules-previous.zip \
       ParameterKey=GuardRulesBucket,UsePreviousValue=true \
       ParameterKey=HookName,UsePreviousValue=true \
       ParameterKey=FailureMode,UsePreviousValue=true \
     --use-previous-template
   ```

2. **For emergency situations**: Temporarily set FailureMode to WARN instead of FAIL:
   ```bash
   aws cloudformation update-stack \
     --stack-name imdsv2-guard-hook \
     --parameters \
       ParameterKey=FailureMode,ParameterValue=WARN \
       ParameterKey=GuardRulesKey,UsePreviousValue=true \
       ParameterKey=GuardRulesBucket,UsePreviousValue=true \
       ParameterKey=HookName,UsePreviousValue=true \
     --use-previous-template
   ```

3. **Complete removal**: In extreme cases, delete the StackSet to remove the hook from all accounts:
   ```bash
   aws cloudformation delete-stack --stack-name imdsv2-hook-org-deployment
   ```

### Exception Handling

Some workloads may legitimately require IMDSv1. To handle exceptions:

1. Create a specific OU for exempted workloads
2. Deploy the hook to all OUs except the exempted one
3. Implement strict access controls for deploying to the exempted OU
4. Document all exceptions with justification and review regularly

## Cost Implications

This solution has minimal cost impact:

1. **CloudFormation Guard Hooks**: No additional charge beyond standard CloudFormation pricing
2. **S3 Storage**: Minimal cost for storing the Guard rules (< 1 MB)
3. **CloudWatch Logs**: Standard CloudWatch pricing for log storage
4. **StackSet Operations**: No additional charges beyond standard CloudFormation pricing

Estimated monthly cost: < $1 for a typical organization

## Security Considerations

1. **IAM Permissions**: The execution role has minimal permissions (S3 read-only for the rules bucket)
2. **S3 Bucket**: Ensure the S3 bucket has appropriate access policies
3. **Hook Execution**: Monitor hook execution logs for potential security issues
4. **StackSet Permissions**: Ensure the StackSet roles have appropriate permissions

## Troubleshooting

If you encounter issues:

1. **Check CloudWatch logs**: Review the CloudWatch log group for the hook
   ```bash
   aws logs get-log-events \
     --log-group-name /aws/cloudformation/hook/Security::EC2::IMDSv2Enforcement \
     --log-stream-name <log-stream-name>
   ```

2. **Review the S3 logs**: If you configured a log bucket, review the detailed Guard evaluation logs

3. **Test locally**: Test your Guard rules locally using the cfn-guard CLI to isolate issues

4. **Common issues**:
   - **Missing permissions**: Ensure the execution role has appropriate S3 permissions
   - **Incorrect rule syntax**: Validate the Guard rules locally before deployment
   - **StackSet deployment failures**: Check the StackSet operations status in the CloudFormation console

## Comprehensive Enforcement Strategy

For a complete IMDSv2 enforcement strategy:

1. **Guard Hook (this repository)**: Preventive control that blocks non-compliant resources
2. **AWS Config Rules**: Detective control to find existing non-compliant resources
3. **SSM Automation**: Remediation for existing non-compliant resources
4. **SCPs**: Organization-level policy enforcement

## Verification

After deployment, attempt to create an EC2 instance or Launch Template without IMDSv2. The operation should fail with a clear error message explaining the requirements.
