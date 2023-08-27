# VPC FlowLogs and Quicksight Dashboards

The CloudFormation Template and Lambda Function have been adapted from the original source [VPC FlowLogs Analysis Dashboard](https://wellarchitectedlabs.com/security/300_labs/300_vpc_flow_logs_analysis_dashboard/) to enable interoperability with AWS Customisations for Control Tower, whilst also ensuring that access to the Log Archive Account is still restricted.

## Architecture Overview

![alt](./Diagrams/FlowLogs.png)

## Pre-Requisites for the Solution
There is an overarching assumption that you already have [Customisation for Control Tower](https://aws.amazon.com/solutions/implementations/customizations-for-aws-control-tower/) deployed within your Control Tower Environment along with the [Security Reference Architecture (SRA)](https://github.com/aws-samples/aws-security-reference-architecture-examples).

### Pre-Requisites for `Centralised FlowLogs Module`
1.  Copy both `centralised-flowlogs-bucket.yaml` and `centralised-flowlogs-shared-bucket.yaml` to the SRA Staging S3 Bucket in the Control Tower Management Account and place them in a prefix named `custom_templates\flowlogs`.
2.  This assumes that an additional AWS Account has already been provisioned for Shared Services.
3.  An S3 Bucket has been created in the Shared Services Account for Athena Query Results to be stored.
4.  Change the URL on Lines 203 and 280 to reflect the correct S3 Staging Bucket created by the Security Reference Architecture.

### Installation for `Centralised FlowLogs Module`
1.  Copy the CloudFormation Template `centralised-flowlogs-module.yaml` to the `/templates` folder for use with Customisations for Control Tower.
2.  Copy the CloudFormation Parameters `centralised-flowlogs-module.json` to the `/parameters` folder for use with Customisations for Control Tower.
3.  Update the CloudFormation Parameters `centralised-flowlogs-module.json` with the required details:
    * **pLogArchiveAccountId:** This is the AWS Account ID of the Account that has been configured by Control Tower as the Log Archive.
    * **pSharedServicesAccountId:** This is the AWS Account ID of the Account that has been configured for Shared Services.
    * **pS3LifeCycleTransition:** This is the number of days before the object published to the S3 Bucket will be transitioned to Amazon S3 Glacier.
    * **pAthenaQueryResultBucketArn:** This is the ARN of the S3 Bucket that will be used in the Shared Services Account for Athena Query Results to be stored.

```json
[
    {
        "ParameterKey": "pLogArchiveAccountId",
        "ParameterValue": ""
    },
    {
        "ParameterKey": "pSharedServicesAccountId",
        "ParameterValue": ""
    },
    {
        "ParameterKey": "pS3LifeCycleTransition",
        "ParameterValue": ""
    },
    {
        "ParameterKey": "pAthenaQueryResultBucketArn",
        "ParameterValue": ""
    }
]
```

4.  Update the `manifest.yaml` and configure the `deployment_targets` accordingly based on your needs. The deployment target should be the AWS Control Tower Management Account.

```yaml
  - name: centralised-flowlogs-dashboards-module
    resource_file: templates/centralised-flowlogs-dashboards-module.yaml
    parameter_file: parameters/centralised-flowlogs-dashboards-module.json
    deploy_method: stack_set
    deployment_targets:
      accounts:
        - # Either the 12-digit Account ID or the Logical Name for the Control Tower Management Account
```

### Pre-Requisites for `Centralised FlowLogs Dashboards Module`
1.  Copy both `centralised-flowlogs-core-infrastructure.yaml` and `centralised-flowlogs-quicksight-dashboards.yaml` to the SRA Staging S3 Bucket in the Control Tower Management Account and place them in a prefix named `custom_templates\flowlogs`.
2.  Ensure that the `Centralised FlowLogs Module` has been successfully deployed.
3.  Ensure that a VPC has been provisioned that has VPC FlowLogs configured and is also configured to send them to the S3 Bucket Created in `centralised-flowlogs-bucket.yaml`.  Once configured wait for some traffic to have been written and confirm that object are appering in the S3 Bucket.
4.  Change the URL on Lines 214 and 293 to reflect the correct S3 Staging Bucket created by the Security Reference Architecture.
5.  Ensure that QuickSight is configured in the Shared Services AWS Account. This includes configuring the Security settings within Quicksight so that the Quicksight Service has permissions to the S3 Bucket provisioned by `centralised-flowlogs-shared-bucket.yaml` as well as the Athena Query Results Bucket and Amazon Athena. Also ensure that the SPICE Capacity has at least 20GB spare, if not provision some more - this can always be freed up later.

### Installation for `Centralised FlowLogs Dashboards Module`

1.  Copy the CloudFormation Template `centralised-flowlogs-dashboards-module.yaml` to the `/templates` folder for use with Customisations for Control Tower.
2.  Copy the CloudFormation Parameters `centralised-flowlogs-dashboards-module.json` to the `/parameters` folder for use with Customisations for Control Tower.
3.  Update the CloudFormation Parameters `centralised-flowlogs-dashboards-module.json` with the required details:
    * **pAthenaQueryResultBucketArn:** This is the ARN of the S3 Bucket that will be used in the Shared Services Account for Athena Query Results to be stored.
    * **pAthenaResultsOutputLocation:** This is the URI of the S3 Bucket that will be used in the Shared Services Account for Athena Query Results to be stored.
    * **pVpcFlowLogsBucketName:** This is the name of the S3 Bucket provisioned by `centralised-flowlogs-shared-bucket.yaml`.
    * **pVpcFlowLogsS3BucketLocation:** This is the URI of the Bucket provisioned by `centralised-flowlogs-shared-bucket.yaml`.
    * **pQuickSightUser:**  This is the Quicksight User that will be the owner of the QuickSight Datasets and Dashboards.
    * **pVpcFlowLogsAthenaDatabase:** This is the name of the Glue Database that the QuickSight Dashboards will be using to query. This should be configured to `vpcflowlogsathenadatabase`.

```json
[
    {
        "ParameterKey": "pLogArchiveAccountId",
        "ParameterValue": ""
    },
    {
        "ParameterKey": "pSharedServicesAccountId",
        "ParameterValue": ""
    },
    {
        "ParameterKey": "pS3LifeCycleTransition",
        "ParameterValue": ""
    },
    {
        "ParameterKey": "pAthenaQueryResultBucketArn",
        "ParameterValue": ""
    }
]
```

4.  Update the `manifest.yaml` and configure the `deployment_targets` accordingly based on your needs. The deployment target should be the AWS Control Tower Management Account.

```yaml
  - name: centralised-flowlogs-module
    resource_file: templates/centralised-flowlogs-module.yaml
    parameter_file: parameters/centralised-flowlogs-module.json
    deploy_method: stack_set
    deployment_targets:
      accounts:
        - # Either the 12-digit Account ID or the Logical Name for the Control Tower Management Account
```

5.  Utilise the following CloudFormation Snippet to ensure that the VPC's are sending there Flowlogs to the centralised log bucket.

```yaml
    Parameters:
        pFlowLogDestinationBucket:
            Type: String
            Description: "The Centralised S3 Bucket for FlowLogs in the Log Archive Account."
    Resources:
        rVpcFlowLog:
            Type: AWS::EC2::FlowLog
            Properties:
            DestinationOptions:
                FileFormat: parquet
                HiveCompatiblePartitions: true
                PerHourPartition: true
            LogDestination: !Sub arn:${AWS::Partition}:s3:::${pFlowLogDestinationBucket}
            LogDestinationType: s3
            LogFormat: ${account-id} ${action} ${az-id} ${bytes} ${dstaddr} ${dstport} ${end} ${flow-direction} ${instance-id} ${interface-id} ${log-status} ${packets} ${pkt-dst-aws-service} ${pkt-dstaddr} ${pkt-src-aws-service} ${pkt-srcaddr} ${protocol} ${region} ${srcaddr} ${srcport} ${start} ${sublocation-id} ${sublocation-type} ${subnet-id} ${tcp-flags} ${traffic-path} ${type} ${version} ${vpc-id}
            MaxAggregationInterval: 600
            ResourceId: !Ref rVpc
            ResourceType: VPC
            TrafficType: ALL
```

6.  When adding the above CloudFormation snippet ensure that you provide the name of the centralised flowlog bucket in the parameter `pFlowLogDestinationBucket`.