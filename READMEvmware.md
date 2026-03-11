# Guidance for Automated Setup of AWS Transform for VMware

## Table of Contents

1. [Overview](#overview)
   - [Architecture](#architecture)
   - [Cost](#cost)
2. [Prerequisites](#prerequisites)
   - [AWS Account](#aws-account)
   - [Supported Regions](#supported-regions)
   - [Software](#software)
3. [Security](#security)
4. [Deployment Steps](#deployment-steps)
5. [Deployment Validation](#deployment-validation)
6. [Running the Guidance](#running-the-guidance)
7. [Troubleshooting](#troubleshooting)
8. [FAQ, known issues, additional considerations, and limitations](#faq-known-issues-additional-considerations-and-limitations)
   - [Service Quotas](#service-quotas)
9. [Cleanup](#cleanup)
10. [Authors](#authors)

## Overview

This Guidance provides an automated approach to deploy AWS Transform for VMware resources using infrastructure as code (IaC), streamlining the setup process by automating the provisioning of required AWS services, network configurations, and security controls. The guidance accelerates time-to-value for organizations migrating and modernizing their VMware workloads while ensuring adherence to AWS best practices and security standards.

The journey begins with a thorough discovery and assessment of your on-premises VMware environment.

1. **AWS Transform for VMware** supports multiple discovery methods:

- RVTools for VMware inventory collection
- [AWS Transform Discovery tool](https://docs.aws.amazon.com/transform/latest/userguide/discovery-tool.html) (Discovery tool), an OVA appliance deployed in your vCenter for gathering network communication patterns between applications and servers.
- Many other discovery data sources are available, see the [discovery documentation](https://docs.aws.amazon.com/transform/latest/userguide/transform-vmware-discover-source-data.html) for details.
  These tools help build a comprehensive view of application-to-application communications, server-to-server dependencies, and overall network topology.

2. **Discovery tool collector** collects crucial network dependency data from your on-premises environment and stores it directly on the OVA appliance. This data forms the foundation for informed migration planning. This data is exported from the appliance and uploaded into AWS Transform. The export format is human-readable, enabling you to inspect your data before sharing it with AWS.

3. **Intelligent network conversion and wave planning**
   With a comprehensive understanding of your environment, AWS Transform for VMware moves to the next critical phase. The Network Conversion Agent automates the creation of AWS CloudFormation templates to set up the target network infrastructure. These templates make sure your cloud environment closely mirrors your on-premises setup, simplifying the setup for the migration.

Meanwhile, the Wave Planning Agent uses advanced graph neural networks to analyze application dependencies and plan optimal migration waves. This minimizes complex portfolio and application dependency analysis, and provides ready-to-migrate wave plans, resulting in smooth migrations.

4. **Enhanced security and compliance**
   Security remains paramount throughout the migration process. AWS Key Management Service (AWS KMS) provides robust encryption for stored data, conversation history, and artifacts when a customer managed key (CMK) is configured. AWS Organizations enables centralized management across multiple AWS accounts, and AWS CloudTrail captures and logs API calls for a complete audit trail.

Access control is managed through AWS Identity and Access Management (IAM), providing centralized access management across AWS accounts. Amazon CloudWatch continuously monitors AWS Transform service activities, resource utilization, and operational metrics within the management account, providing full visibility and control throughout the migration process.

5. **Orchestrated migration execution**
   When it’s time to execute the migration, the Migration Agent orchestrates the migration and cutover process. It works in tandem with AWS Application Migration Service to replicate source servers to Amazon Elastic Compute Cloud (Amazon EC2) instances based on the carefully planned waves and groupings.

The AWS Provisioning/Target Account serves as the production environment where your migrated applications will reside. This account contains the target infrastructure and will house your production workloads after migration is complete. S3 buckets in this account store the CloudFormation templates used for infrastructure deployment, providing a smooth, consistent, and reliable setup process.

6. **Flexible network configuration**
   AWS Transform for VMware offers two networking models to suit different requirements:

- Hub-and-spoke model – AWS Transit Gateway connects virtual private clouds (VPCs) through a central hub VPC with shared NAT gateways. This model is ideal for centralized management and shared services.
- Isolated model – Each VPC operates independently, connected directly by Transit Gateway. This approach offers greater isolation and is suitable for environments with strict separation requirements.

VPCs created by AWS Transform match your on-premises network segments, providing a seamless transition. NAT gateways provide outbound internet access for private subnets, maintaining security while enabling necessary connectivity. In hub-and-spoke deployments, shared NAT gateways are used in the central hub VPC, whereas in isolated deployments, individual NAT gateways are created for each VPC.

### Architecture

Below is the Reference architecture for the guidance showing the core and supporting AWS services:

![architecture](assets/aws_transform_vmware_ref-arch1.png)

**Figure 1. Automated Setup of AWS Transform for VMware - environment setup and access configuration**

1. Customer VMware environment hosts the workloads to be migrated. RVTools can be used along with optional import/export functionality for customers running VMware NSX.
2. AWS Transform discovery tool (instead of RVTools) gathers and collects data and dependencies for migration. AWS Replication Agent migrates virtual machines to AWS.
3. AWS Transform for VMware workspaces are available globally.
   A full list of supported AWS Regions can be found [here](https://docs.aws.amazon.com/transform/latest/userguide/regions.html).
4. AWS Transform for VMware helps optimize infrastructure and reduce operational overhead, giving you a more predictable, cost-efficient path to modernization.
5. As part of AWS Transform, the Wave Planning capability uses graph neural networks to analyze application dependencies and plan migration waves. AWS Transform utilizes an AI-powered migration planning capability with a conversational interface to transform complex VMware infrastructure data into actionable migration strategies through intelligent dependency analysis and structured validation.



![architecture2](assets/aws_transform_vmware_ref-arch2.png)

**Figure 2. Automated Setup of AWS Transform for VMware - data collection and initial migration planning**

6. The AWS migration planning account hosts AWS Transform for migration planning activities.
7. AWS Key Management Service (KMS) encrypts data using AWS managed keys by default or optional Customer Managed Keys (CMK).
8. AWS Organizations enables centralized management of AWS accounts through Organizational Units.
9. Amazon CloudWatch monitors AWS Transform activities, resources, and metrics in the management account.
10. AWS Identity and Access Management (IAM) Identity Center provides centralized access management across all AWS accounts.
11. Amazon S3 buckets store key migration artifacts including inventory data, dependency mappings, wave plans, and application groupings. The artifact store is located in the AWS Transform service account. Discovery data and migraiton artifcats are also stored there.
12. AWS CloudFormation automates resource provisioning across AWS accounts and regions for test and production environments.
13. AWS CloudTrail logs API activities in AWS accounts, while AWS Transform service tracks migration activities.

![architecture3](assets/aws_transform_vmware_ref-arch3.png)

**Figure 3. Automated Setup of AWS Transform for VMware - workload migration and network conversion to AWS**

 **NOTE**: For the most up-to-date information on supported Regions, please refer to [AWS Services by Region](https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/)

14. The AWS Target/Provisioning Account hosts migrated production workloads and applications.
15. The AWS Transform network migration capability converts on-premises networks to AWS using CloudFormation, AWS Cloud Development Kit (AWS CDK) templates, Terraform and Amazon Landing Zone Accelerator are also supported.
16. AWS Transform orchestrates end-to-end migration by coordinating across various AWS tools and service, including Server Migration/Rehost capability utilizing AWS Application Migration Service.
17. Amazon Elastic Compute Cloud (EC2) and Amazon Elastic Block Store (EBS) host migrated VMware virtual machines with recommended instance types
18. The network foundation of this migration architecture relies on Amazon Virtual Private Cloud (VPC) and Amazon Transit Gateway working in tandem, where VPC provides dedicated network isolation for migrated workloads while Transit Gateway acts as the central hub connecting these VPCs, with Amazon NAT Gateways enabling secure internet access for private subnet resources. AWS Application Migration Service handles the core migration execution by managing both the initial server replication process and orchestrating the test/cutover instance launches, being supported by a comprehensive set of AWS services (KMS, CloudWatch, CloudTrail, IAM permissions, CloudFormation, and AWS S3) that work together to maintain security, enable in-depth monitoring, and automate the infrastructure deployment through stored per-wave migration plans.

### Cost

While this guidance provides default configurations, customers are responsible for:

1. Configuring the guidance to their optimal settings based on their specific use case and requirements
2. Monitoring and managing the costs incurred from any supporting AWS services
3. Reviewing and optimizing storage usage for transformation artifacts

We recommend creating a [Budget](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html) through [AWS Cost Explorer](https://aws.amazon.com/aws-cost-management/aws-cost-explorer/) to help manage costs. Prices are subject to change. For full details, refer to the pricing webpage for each AWS service used in this Guidance.

As of April 2024, the cost for running this Guidance with the default settings in the default AWS Region (US East - N. Virginia) us-east-1 is the following:

| AWS Service             | Dimensions                         | Cost [USD]              |
| ----------------------- | ---------------------------------- | ----------------------- |
| AWS Transform           | Unlimited transformations          | Free                    |
| AWS IAM Identity Center | Number of users                    | Free                    |
| AWS Organizations       | Number of accounts in organization | Free                    |
| AWS Lambda              | First 1M requests                  | Free                    |
| AWS Lambda              | Over 1M requests                   | $0.20 per 1M requests   |
| Amazon S3               | Standard Storage                   | $0.023 per GB per month |
| AWS KMS                 | Customer managed key (optional)    | $1.00 per key per month |

## Prerequisites

### AWS Account

An AWS Account with [IAM admin access](https://docs.aws.amazon.com/streams/latest/dev/setting-up.html) is required to run the scripts that will enable AWS organization and the Identity Center groups.

### Supported Regions

For the full list of supported regions view [AWS Transform Regions](https://docs.aws.amazon.com/transform/latest/userguide/regions.html).
Currently, the following regions are supported by default:

- US East (N. Virginia)
- Europe (Frankfurt)
- Asia Pacific (Mumbai)
- Asia Pacific (Sydney)
- Asia Pacific (Tokyo)
- Europe (London)
- Asia Pacific (Seoul)
- Canada (Central)

### Software

- The machine from which this guidance is deployed needs to support Linux BASH or PowerShell scripts. Alternatively, the parameters can be manually added to the CloudFormation YAML files.
- Source code repository (GitHub, GitLab, or Bitbucket)
- Visual Studio IDE (for IDE integration)

## Security

When you build systems on AWS infrastructure, security responsibilities are shared between you and AWS. This [shared responsibility model](https://aws.amazon.com/compliance/shared-responsibility-model/) reduces your operational burden because AWS operates, manages, and controls the components including the host operating system, the virtualization layer, and the physical security of the facilities in which the services operate. For more information about AWS security, visit [AWS Cloud Security](https://aws.amazon.com/security/).

Users should refer to the [AWS Transform security documentation](https://docs.aws.amazon.com/transform/latest/userguide/security.html) to understand the full scope of permissions that may be required for their particular use cases and transformation scenarios. Key security considerations include source code security (secure access to repositories, encryption of code during transformation, protection of transformation artifacts), identity and access management (role-based access control through IAM Identity Center, least privilege access principles, separation of duties between administrators and users), and data protection (encryption at rest for transformation artifacts, secure transmission of code and artifacts, optional customer-managed KMS keys for additional control).

## Deployment Steps

### Clone Guidance repository

1. Log in to your AWS account on your CLI/shell through your preferred authentication provider.

2. Clone the repository:

   ```bash
   git clone https://github.com/aws-solutions-library-samples/guidance-for-automated-setup-of-aws-transform
   ```

### Phase 1: Set up AWS Organizations

> Note : If you already have AWS Organizations enabled in your Management account, you can skip Phase 1.

1. Change directory to the source folder inside the guidance repository:

   ```bash
   cd guidance-for-automated-setup-of-aws-transform/source
   ```

2. Start by running the first shell script. This creates an AWS Organization with all features enabled.

   - STACK_NAME: {name of CloudFormation stack}.
     PLATE_PATH: {path to `phase2-idc.yaml`}.

   BASH:

   ```bash
   source % ./deploy-phase1.sh
   Enter stack name [aws-org-setup]: aws-org-setup
   Enter template path [/guidance-for-automated-setup-of-aws-transform/source/phase1-aws-organizations.yaml]:
   ```

   PowerShell:

   ```powershell
       PS C:\git\aws\guidance-for-automated-setup-of-aws-transform\source> .\deploy-phase1.ps1
       Enter stack name [aws-org-setup]:
       Enter template path [phase1-aws-organizations.yaml]:
   ```

   > Note : A Powershell script is available for Windows OS. Alternatively, the parameters can be manually added to the CloudFormation YAML.

3. After successful deployment, you will need to manually enable an organization instance of IAM Identity Center in the AWS Console (wait a few minutes for the changes to propagate), as shown below:

![enable_identity_center](assets/enable_identity_center.png)

**Figure 4. Enable an Organization instance of IAM Identity Center**

### Phase 2: Set up IAM Identity Center

1. After enabling IAM Identity Center manually and waiting for updates to propagate, run the second BASH script

   Pass in the following parameters using selected script:

   - STACK_NAME: {name of cloudformation stack}.
   - TEMPLATE_PATH: {path to phase2 yaml}.
   - ACCOUNT_NUMBER: {AWS account number}.
   - IDENTITY_CENTER_ID: {AWS Identity Center ID}.
   - ADMIN_EMAIL: {Email for admin user provisioned by script}.

   BASH script:

   ```bash
       source % ./deploy-phase2.sh
       Enter stack name [aws-transform-setup]:
       Enter template path: [/guidance-for-automated-setup-of-aws-transform/source/phase2-idc.yaml]:
       Enter AWS account number: 1234567XXXXXX
       Enter admin email address: admin@amazon.com
       Enter Identity Center ID: ssoins-1234a123b1d5ab3f
       Retrieving Identity Store ID for IAM Identity Center instance ssoins-1234a252c3d5bd2f...
       Found Identity Store ID: d-40338374bc
   ```

   PowerShell:

   ```powershell
       PS C:\git\aws\guidance-for-automated-setup-of-aws-transform\source> .\deploy-phase2.ps1
       Enter stack name [aws-transform-setup]:
       Enter template path [phase2-idc.yaml]:
       Enter AWS account number: 1234567XXXXXXX
       Enter admin email address: admin@amazon.com
       Enter Identity Center ID: ssoins-1234a123b1d5ab3f
       Retrieving Identity Store ID for IAM Identity Center instance ssoins-1234a252c3d5bd2f...
       Found Identity Store ID: d-40338374bc
   ```

   This CloudFormation will:

- Create IAM Identity Center groups and users
- Set up the necessary IAM policies for AWS Transform for VMware for both groups
- Create an Admin user using lambda functions in Identity Center based on a provided email

> Note: The script/CloudFormation uses the deployed Lambda functions to add the provided email account as an Admin in the created AWS Transform Admin group in AWS IAM Identity Center. Subsequent admins and users can be added via the AWS console following best practices.

## Deployment Validation

- Open CloudFormation in AWS console and verify the status of the stacks

![CloudFormation Stack Status](assets/cfn_stack.png)

**Figure 5. Guidance Cloud Formation Stack Deployment Status**

- Open Identity Center and verify the created groups:

![Identity Center Groups](assets/idc_group.png)

**Figure 6. Verify Identity Center Groups**

- Open Identity Center and select Multi Account Permissions → AWS accounts. Select Assign Users or Groups

![IDC AWS Accounts](assets/idc_awsaccounts.png)

**Figure 7. IDC AWS Accounts Select Assign Users or Groups**

- Select Admin IDC Group

![Select Group](assets/select_group.png)

**Figure 8. Select Admin IDC Group**

- Select Admin Permission Set

![Select Permission Set](assets/select_set.png)

**Figure 9. Select Admin IDC Permission Set**

- Submit. Repeat for User group/permission set.

![Review and Submit](assets/review_and_submit.png)

**Figure 10. Submit. Repeat for User IDC Group and Permission Set**

- Review the admin group and verify created user

![Admin User](assets/admin_user.png)

**Figure 11. View the Administrators Group and Verify Created User**

- Make sure the groups can be added to AWS Transform

![Transform Group](assets/transform_group.png)

**Figure 12. Verify that IDC groups can be added to AWS Transform**

- Make sure the start URL can be accessed by Admin user

![Transform Start](assets/transform_start.png)

**Figure 13. Verify that Start URL can be accessed by Administrator User**

At this point all of the pre-requisites are complete and you are ready now to use AWS Transform for VMware. Follow the steps [here](https://docs.aws.amazon.com/transform/latest/userguide/what-is-transform.html) to continue

## Running the Guidance

Please see the [AWS official documentation](https://docs.aws.amazon.com/transform/latest/userguide/what-is-transform.html) of AWS Transform for VMware for details of using the Service.

Before starting your own VMware modernization project, we recommend completing the [AWS Transform for VMware workshop](https://catalog.workshops.aws/transform-vmware). In this workshop, you can gain hands-on experience with configuration, planning and assessment of VMware migration using AWS Transform. 

Please feel free to explore our [self-guided demo](https://app.storylane.io/share/ulhid0hbuxvd) to learn how AWS Transform for VMware accelerates large-scale migration of VMware based workloads to AWS.

### Troubleshooting

In case when either of the guidance deployment phases described above fail, you should start troubleshooting their deployment from the CloudFormation console that would show the failed step in the "Events" tab as illustrated in the example below:

![Phase 2 Deployment Failed](assets/phase2_stack_creation_failed.jpg)

**Figure 14. Example of failed phase 2 Cloud Formation deployment**

To determine the root cause, follow the specified log group in the Cloud Watch service area from the error message and locate an event that contains an Error log message as illustrated below:

![Phase 2 CloudWatch Log](assets/phase2_stack_creation_failed_cloudwatch_log_details.jpg)

**Figure 15. Example of CloudWatch log with a message for failed phase 2 Cloud Formation deployment**

Then you can examine the detailed message and determine the root cause of an error:

```
"Data": {
        "Error": "An error occurred (AccessDeniedException) when calling the CreateGroupMembership operation: User: arn:aws:sts::1234567XXXXXXX:assumed-role/aws-transform-setup-IdentityCenterLambdaRole-WriIgQEV4Wx9/aws-transform-setup-AddUserToGroupFunction-Z1ugUVrdHWr7 is not authorized to perform: identitystore:CreateGroupMembership on resource: arn:aws:identitystore:::group/6468d408-50b1-7045-2426-80200c9a324f because no identity-based policy allows the identitystore:CreateGroupMembership action, User: arn:aws:sts::354918380621:assumed-role/aws-transform-setup-IdentityCenterLambdaRole-WriIgQEV4Wx9/aws-transform-setup-AddUserToGroupFunction-Z1ugUVrdHWr7 is not authorized to perform: identitystore:CreateGroupMembership on resource: arn:aws:identitystore:::user/f458b4c8-f081-7031-262c-791e8a173a98 because no identity-based policy allows the identitystore:CreateGroupMembership action"
    }
```
and make necessary updates. In this example the issue is resolved by adding necessary IAM permission policy.

## FAQ, known issues, additional considerations, and limitations

### Service Quotas

AWS Transform enforces specific service quotas, please review [AWS Transform Service Quotas](https://docs.aws.amazon.com/transform/latest/userguide/transform-limits.html) for more information.

## Cleanup

When you no longer need to use the guidance, you should delete the AWS resources deployed in order to prevent ongoing charges for their usage.

In the AWS Management Console, navigate to CloudFormation and locate the 2 guidance stacks deployed (typcally named `aws-org-setup` and `aws-transform-setup` as shown in the **Deployment** section above). Starting with the most recent stack (not including any nested stacks), select it and click `Delete` button:

![cleanupcfn](assets/cleanup_cfn.png)

When both stacks are successfully deleted, the corresponding AWS resources should be deleted as well.

## Authors

- Pranav Kumar, GenAI Labs Builder SA
- Johanna Wood, Technical Domain Lead, 
- Daniel Zilberman, Sr. Specialist SA, AWS Tecnhical Solutions
- Deepika Suresh, Specialist SA, AWS Tecnhical Solutions

