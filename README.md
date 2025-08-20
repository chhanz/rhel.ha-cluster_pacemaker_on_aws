# rhel.ha-cluster_pacemaker_on_aws

A solution for automatically configuring RHEL-based High Availability (HA) clusters on AWS.

## For Korean
This README is also available in Korean. Please use the Korean version, [`README_KR.md`](README_KR.md).

## Overview

This project consists of two main components:
1. **CloudFormation Template** (`deployment.yaml`) - AWS infrastructure configuration
2. **SSM Document** (`ha-cluster-setup-document.json`) - Automated HA cluster setup

## Architecture

### Infrastructure Configuration
- **VPC**: 10.0.0.0/16 CIDR block
- **Subnets**: 
  - SubnetB (ap-northeast-2b): 10.0.1.0/24
  - SubnetD (ap-northeast-2d): 10.0.2.0/24
- **EC2 Instances**: 2x RHEL 9.6 (t3.medium)
- **Security Groups**: Allow internal cluster communication
- **IAM Roles**: STONITH and Elastic IP management permissions

### HA Cluster Configuration
- **Pacemaker/Corosync** based cluster
- **STONITH**: Using `fence_aws`
- **Ansible**: Automated configuration with rhel-system-roles
- **Optional Dummy Resource**: For testing purposes

## Prerequisites

- AWS CLI installed and configured
- Appropriate AWS permissions (EC2, IAM, SSM)
- Seoul region (ap-northeast-2) usage

## Installation and Usage

### Step 1: Infrastructure Deployment

```bash
# Create CloudFormation stack
aws cloudformation create-stack \
    --stack-name rhel-ha-cluster \
    --template-body file://deployment.yaml \
    --parameters ParameterKey=SameSubnet,ParameterValue=false \
    --capabilities CAPABILITY_NAMED_IAM \
    --region ap-northeast-2
```

#### Parameters
- `SameSubnet`: 
  - `false` (default): Deploy in different AZs
  - `true`: Deploy in the same subnet

### Step 2: Create SSM Document

```bash
# Create SSM document
aws ssm create-document \
    --name "HA-Cluster-Setup" \
    --document-type "Command" \
    --document-format "JSON" \
    --content file://ha-cluster-setup-document.json \
    --region ap-northeast-2
```

### Step 3: Configure HA Cluster

***[CAUTION] HA cluster setup execution performs SSM document "Run Command" only on Node 1.***   
   
```bash
# Check instance information
aws cloudformation describe-stacks \
    --stack-name rhel-ha-cluster \
    --query 'Stacks[0].Outputs' \
    --region ap-northeast-2

# Execute HA cluster setup > Run SSM document command on one instance to be used as Node 1
aws ssm send-command \
    --document-name "HA-Cluster-Setup" \
    --instance-ids "i-xxxxxxxxx" \
    --parameters '{
        "Node1InstanceId":"i-xxxxxxxxx",
        "Node1PrivateIP":"10.0.1.10",
        "Node2InstanceId":"i-yyyyyyyyy",
        "Node2PrivateIP":"10.0.2.20",
        "ClusterPassword":"secure-password",
        "ClusterName":"my-ha-cluster",
        "DeployDummyResource":"false"
    }' \
    --region ap-northeast-2
```

## Parameter Description

### CloudFormation Parameters
| Parameter | Default | Description |
|-----------|---------|-------------|
| SameSubnet | false | Instance placement method selection |

### SSM Document Parameters
| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| Node1InstanceId | ✓ | - | Node1 instance ID |
| Node1PrivateIP | ✓ | - | Node1 private IP |
| Node2InstanceId | ✓ | - | Node2 instance ID |
| Node2PrivateIP | ✓ | - | Node2 private IP |
| ClusterPassword | | redhat | Cluster password |
| ClusterName | | fast-aws-rh-cluster | Cluster name |
| DeployDummyResource | | false | Deploy test dummy resource |

## Key Features

### Automated Configuration
- **System Updates**: Automatic update to latest packages
- **User Creation**: Automatic haadm account creation
- **Connect the Instance **: Configure to use Session Manager
- **Required Packages**: Install rhel-system-roles, AWS CLI, and other tools

### HA Cluster Setup
- **Firewall/SELinux**: Automatic deactivation
- **Boot Start**: Disabled (manual start)
- **Cloud Agent**: Automatic installation
- **STONITH Configuration**: Using `fence_aws`

### Network Configuration
- **Host File**: Automatic update (/etc/hosts)
- **Ansible Inventory**: Dynamic generation

## File Structure

```
rhel.ha-cluster_pacemaker_on_aws/
├── deployment.yaml                 # CloudFormation template
├── ha-cluster-setup-document.json  # SSM document
├── README.md                       # This file
└── README_KR.md                    # README for Korean
```

## Generated Files

When executing the SSM document, the following files are created in `/usr/local/ha_cluster/`:

- `inventory.yml`: Ansible inventory
- `update-hosts.yaml`: Host file update playbook
- `fast-aws-playbook.yaml`: HA cluster deployment playbook
- `group_vars/${CLUSTER_NAME}.yml`: Cluster configuration variables

## Security Considerations

- **IAM Permissions**: Apply principle of least privilege
- **Security Groups**: Allow only VPC internal communication
- **IMDS**: Enhanced security using v2

## Troubleshooting

### Common Issues
1. **SSM Agent Connection Failure**: Check IAM role
2. **Ansible Execution Failure**: Check SSH connectivity
3. **STONITH Failure**: Check IAM permissions

### Log Verification
```bash
# Check SSM command execution status
aws ssm get-command-invocation \
    --command-id "command-id" \
    --instance-id "i-xxxxxxxxx" \
    --region ap-northeast-2

# Check cluster status
sudo pcs status
sudo pcs config
```

## Cleanup

```bash
# Delete CloudFormation stack
aws cloudformation delete-stack \
    --stack-name rhel-ha-cluster \
    --region ap-northeast-2

# Delete SSM document
aws ssm delete-document \
    --name "HA-Cluster-Setup" \
    --region ap-northeast-2
```

## Contributing

Please submit bug reports or feature requests through issues.