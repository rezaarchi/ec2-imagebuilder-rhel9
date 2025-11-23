# RHEL 9 STIG AMI Builder with EC2 Image Builder

Automated solution for creating DISA STIG-compliant Red Hat Enterprise Linux 9 AMIs using AWS EC2 Image Builder, CloudFormation, and Ansible.

## Overview

This project automates the creation of Security Technical Implementation Guide (STIG) hardened RHEL 9 Amazon Machine Images (AMIs) that meet federal compliance requirements for FedRAMP, NIST 800-53, and DISA STIG standards.

### Key Features

- **Automated STIG Compliance**: Applies the official Red Hat RHEL 9 STIG Ansible role (RedHatOfficial.rhel9_stig)
- **Encrypted Storage**: All EBS volumes encrypted with customer-managed KMS keys
- **Image Builder Safe**: Custom implementation handles firewalld and authselect in EC2 Image Builder environments
- **Repeatable Builds**: Infrastructure-as-Code approach ensures consistent, auditable AMI creation
- **First-Boot Configuration**: Systemd unit enables firewalld and SSH access on first instance launch

### STIG Controls Implemented

The solution implements comprehensive DISA STIG controls including:
- File permissions and ownership hardening
- System auditing configuration (auditd)
- Password policy enforcement
- Network security settings
- FIPS 140-2 cryptographic module configuration
- User authentication with authselect custom profiles
- Intrusion detection with AIDE (Advanced Intrusion Detection Environment)

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     CloudFormation Stack                     │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐      ┌──────────────────────────────┐    │
│  │ Image Builder│──────▶│  Infrastructure Config       │    │
│  │   Pipeline   │      │  - Instance Profile          │    │
│  └──────────────┘      │  - Security Groups           │    │
│         │              │  - KMS Key                   │    │
│         │              └──────────────────────────────┘    │
│         ▼                                                    │
│  ┌──────────────────────────────────────────────────┐      │
│  │            Image Recipe (RHEL 9)                 │      │
│  │  ┌────────────────────────────────────────────┐ │      │
│  │  │ 1. Install Ansible + STIG Role Component  │ │      │
│  │  │    - Ansible Core & Collections           │ │      │
│  │  │    - RedHatOfficial.rhel9_stig            │ │      │
│  │  │    - AIDE, FIPS, SSSD packages            │ │      │
│  │  │    - Authselect custom profile            │ │      │
│  │  └────────────────────────────────────────────┘ │      │
│  │  ┌────────────────────────────────────────────┐ │      │
│  │  │ 2. Apply RHEL 9 STIG Component            │ │      │
│  │  │    - Run Ansible playbook                 │ │      │
│  │  │    - Configure first-boot firewalld unit  │ │      │
│  │  └────────────────────────────────────────────┘ │      │
│  └──────────────────────────────────────────────────┘      │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────┐                                           │
│  │ STIG-Hardened│                                           │
│  │  RHEL 9 AMI  │                                           │
│  └──────────────┘                                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Prerequisites

Before deploying this solution, ensure you have:

- **AWS Account** with appropriate permissions
- **AWS CLI** configured with credentials
- **VPC** with at least one subnet
- **NAT Gateway or Internet Gateway** for outbound internet access
- **EC2 Key Pair** named `imagebuilder` (or modify the template)
- **IAM Permissions** to create:
  - CloudFormation stacks
  - EC2 Image Builder resources
  - IAM roles and policies
  - S3 buckets
  - KMS keys
  - Security groups

## Installation

### 1. Clone the Repository

```bash
git clone https://github.com/rezaarchi/ec2-imagebuilder-rhel9.git
cd ec2-imagebuilder-rhel9
```

### 2. Deploy the CloudFormation Stack

#### Option A: AWS CLI

```bash
aws cloudformation create-stack \
  --stack-name rhel9-stig-imagebuilder \
  --template-body file://IBM_2.yml \
  --parameters \
    ParameterKey=VpcId,ParameterValue=vpc-xxxxxxxxx \
    ParameterKey=DemoSubnetIds,ParameterValue=subnet-xxxxxxxxx \
    ParameterKey=STIGArtifactsBukcet,ParameterValue=my-stig-artifacts-bucket \
    ParameterKey=BuildInstanceType,ParameterValue=t3.medium \
  --capabilities CAPABILITY_IAM \
  --region us-east-1
```

#### Option B: AWS Console

1. Navigate to **CloudFormation** in the AWS Console
2. Click **Create Stack** → **With new resources**
3. Upload the `IBM_2.yml` template file
4. Fill in the required parameters (see Parameter Reference below)
5. Acknowledge IAM resource creation
6. Click **Create Stack**

### 3. Monitor Stack Creation

```bash
# Check stack status
aws cloudformation describe-stacks \
  --stack-name rhel9-stig-imagebuilder \
  --query 'Stacks[0].StackStatus' \
  --output text

# Watch stack events
aws cloudformation describe-stack-events \
  --stack-name rhel9-stig-imagebuilder \
  --max-items 10
```

### 4. Trigger the Image Build

Once the stack is created (status: `CREATE_COMPLETE`), manually trigger the pipeline:

```bash
# Get the pipeline ARN
PIPELINE_ARN=$(aws cloudformation describe-stack-resources \
  --stack-name rhel9-stig-imagebuilder \
  --logical-resource-id ImagePipeline \
  --query 'StackResources[0].PhysicalResourceId' \
  --output text)

# Start the image build
aws imagebuilder start-image-pipeline-execution \
  --image-pipeline-arn $PIPELINE_ARN
```

### 5. Monitor the Build Process

```bash
# List recent image builds
aws imagebuilder list-image-pipeline-images \
  --image-pipeline-arn $PIPELINE_ARN \
  --max-results 5

# Check build logs in S3
aws s3 ls s3://YOUR-LOG-BUCKET-NAME/imagebuilder-rhel9-stig-imagebuilder/ --recursive
```

The build process typically takes **30-60 minutes** to complete.

## Parameter Reference

| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| **VpcId** | AWS::EC2::VPC::Id | Yes | VPC where build instances will launch. Must have internet connectivity via NAT/IGW. | `vpc-0123456789abcdef0` |
| **DemoSubnetIds** | AWS::EC2::Subnet::Id | Yes | Subnet for build instances. Should be private with NAT Gateway for outbound access. | `subnet-0123456789abcdef0` |
| **STIGArtifactsBukcet** | String | Yes | S3 bucket name for STIG artifacts. Will be created if it doesn't exist. Must be globally unique. | `my-org-stig-artifacts-2024` |
| **BuildInstanceType** | String | No | EC2 instance type for builds. Default: `t3.medium`. Larger instances speed up builds but increase costs. | `t3.medium`, `t3.large`, `t2.medium` |

## Usage

### Launching Instances from the STIG AMI

Once the build completes, launch EC2 instances using the new AMI:

```bash
# Get the AMI ID
AMI_ID=$(aws ec2 describe-images \
  --owners self \
  --filters "Name=name,Values=rhel9-stig-image-recipe*" \
  --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
  --output text)

# Launch an instance
aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.medium \
  --key-name your-key-pair \
  --subnet-id subnet-xxxxxxxxx \
  --security-group-ids sg-xxxxxxxxx \
  --iam-instance-profile Name=YourInstanceProfile \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=RHEL9-STIG-Instance}]'
```

### First Boot Behavior

On first boot, the instance automatically:
1. Enables and starts `firewalld`
2. Opens SSH port (22) in the firewall
3. Creates a marker file to prevent re-running

This is handled by the `firstboot-firewalld.service` systemd unit.

### Verifying STIG Compliance

Connect to your instance and verify STIG configuration:

```bash
# Check authselect profile
authselect current

# Verify AIDE installation
rpm -q aide
aide --check

# Check firewalld status
systemctl status firewalld
firewall-cmd --list-all

# Review auditd rules
auditctl -l

# Check FIPS mode
fips-mode-setup --check
```

## Architecture Components

### S3 Buckets

- **ImageBuilderLogBucket**: Stores Image Builder execution logs
  - Versioning enabled
  - Server-side encryption (AES256)
  - Public access blocked
  - SSL-only access enforced
  
- **STIGArtifactsBucket**: Stores STIG configuration artifacts

### IAM Resources

- **InstanceRole**: IAM role for build instances
  - Managed policies: `AmazonSSMManagedInstanceCore`, `EC2InstanceProfileForImageBuilder`
  - Custom policies for SSM, Image Builder, and CloudWatch Logs
  
- **InstanceProfile**: Instance profile attached to build instances

### Security

- **KMS Key**: Customer-managed key for EBS encryption
  - Automatic key rotation enabled
  - Used for AMI and snapshot encryption
  
- **Security Group**: Build instance security group
  - Allows all outbound traffic (for package downloads)
  - No inbound rules (SSM Session Manager for access)

### Image Builder Components

1. **InstallAnsibleSTIGRoleComponent** (v1.1.14)
   - Installs Ansible and required collections
   - Installs RedHatOfficial.rhel9_stig role
   - Configures AIDE, FIPS, SSSD, and authselect
   - Creates STIG playbook
   - Sets up first-boot firewalld systemd unit

2. **ApplyRHEL9STIGComponent** (v1.1.14)
   - Runs Ansible playbook in check mode
   - Applies STIG configuration
   - Verifies configuration

### Image Recipe

- **Base Image**: Red Hat Enterprise Linux 9 (latest)
- **Block Device**: 20 GB gp3 EBS volume (encrypted)
- **SSM Agent**: Remains installed after build

## Cost Considerations

**IMPORTANT:** Running this solution will incur AWS charges. Estimated cost is **$5-15/month** for occasional builds, primarily from EC2 build instances (~$0.04/hour during 30-60 min builds), EBS snapshots (~$1/month per AMI), and KMS keys ($1/month). To minimize costs, delete unused AMIs and snapshots, disable the pipeline when not in use, and set up AWS billing alerts.

### Cost Breakdown

| Resource | Cost | Frequency |
|----------|------|-----------|
| EC2 Build Instance (t3.medium) | ~$0.04/hour | Per build (~1 hour) |
| EBS Snapshot Storage | ~$0.05/GB/month | Per AMI retained |
| KMS Key | $1/month | Ongoing |
| S3 Storage | ~$0.023/GB/month | Minimal |

### Cost Optimization Tips

- **Delete old AMIs**: Remove AMIs and associated snapshots when no longer needed
  ```bash
  aws ec2 deregister-image --image-id ami-xxxxxxxxx
  aws ec2 delete-snapshot --snapshot-id snap-xxxxxxxxx
  ```
  
- **Disable pipeline**: Set pipeline status to `DISABLED` when not actively building
  ```bash
  aws imagebuilder update-image-pipeline \
    --image-pipeline-arn $PIPELINE_ARN \
    --status DISABLED
  ```

- **Use lifecycle policies**: Configure S3 lifecycle rules to transition logs to cheaper storage tiers

- **Set up billing alerts**: Create CloudWatch billing alarms in the AWS Billing Console

## Troubleshooting

### Build Failures

**Check Image Builder logs:**
```bash
# List executions
aws imagebuilder list-image-build-versions \
  --image-version-arn $IMAGE_VERSION_ARN

# Download logs from S3
aws s3 sync s3://YOUR-LOG-BUCKET/imagebuilder-rhel9-stig-imagebuilder/ ./logs/
```

**Common issues:**
- **No internet access**: Ensure subnet has NAT Gateway and route to internet
- **SSM connection failed**: Verify IAM role has `AmazonSSMManagedInstanceCore` policy
- **Ansible role installation failed**: Check outbound HTTPS access to `galaxy.ansible.com`
- **AIDE initialization failed**: May need more time or larger instance type

### Stack Creation Failures

**Invalid bucket policy:**
- Ensure bucket names are unique and follow S3 naming conventions
- Verify IAM permissions are sufficient

**Key pair not found:**
- Create an EC2 key pair named `imagebuilder` or modify the template

**VPC/Subnet issues:**
- Ensure VPC ID and Subnet ID are valid and in the same region

### Instance Launch Issues

**First boot firewalld not running:**
```bash
# Check service status
systemctl status firstboot-firewalld.service

# View logs
journalctl -u firstboot-firewalld.service

# Manually enable if needed
systemctl enable firewalld
systemctl start firewalld
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload
```

**STIG compliance check failures:**
```bash
# Re-run Ansible playbook
ansible-playbook /tmp/rhel9-stig-playbook.yml

# Check for errors
grep -i error /var/log/messages
```

## Compliance and Security

### STIG Coverage

This solution implements controls from:
- **DISA STIG for RHEL 9**: Latest version from Red Hat Official
- **NIST 800-53**: Access Control, Audit and Accountability, Configuration Management, Identification and Authentication, System and Communications Protection
- **FedRAMP**: Moderate and High baseline controls

### Security Best Practices

- All EBS volumes encrypted with customer-managed KMS keys
- S3 buckets enforce SSL/TLS and encryption
- Least privilege IAM policies
- No public internet access for build instances (NAT Gateway egress only)
- Systems Manager Session Manager for secure access (no SSH keys in logs)

### Compliance Validation

Validate compliance using:
- **OpenSCAP**: Run SCAP scans against the RHEL 9 STIG profile
- **AWS Inspector**: Enable Inspector for vulnerability scanning
- **Custom scripts**: Automated compliance checking in your CI/CD pipeline

```bash
# Example OpenSCAP scan
oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_stig \
  --results scan-results.xml \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

## Contributing

Contributions are welcome! Please follow these guidelines:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/your-feature`)
3. Test your changes thoroughly
4. Update documentation as needed
5. Submit a pull request

### Reporting Issues

- Use GitHub Issues for bug reports and feature requests
- Include relevant logs and error messages
- Describe steps to reproduce the issue

## License

Copyright 2025 IBM.com. All Rights Reserved.

Author: Reza Beykzadeh

## Additional Resources

- [AWS EC2 Image Builder Documentation](https://docs.aws.amazon.com/imagebuilder/)
- [Red Hat RHEL 9 STIG Role](https://galaxy.ansible.com/RedHatOfficial/rhel9_stig)
- [DISA STIGs](https://public.cyber.mil/stigs/)
- [NIST 800-53 Controls](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)
- [FedRAMP Program](https://www.fedramp.gov/)

## Support

For questions or support:
- Open an issue in GitHub
- Contact: Reza Beykzadeh (IBM Federal)

## Acknowledgments

- Red Hat for the official RHEL 9 STIG Ansible role
- AWS EC2 Image Builder team
- DISA for maintaining STIG requirements
