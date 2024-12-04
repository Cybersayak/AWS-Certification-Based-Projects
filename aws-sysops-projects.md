# AWS SysOps Administrator Practice Projects

## Project 1: Advanced Monitoring and Alerting System
### Learning Objectives:
- CloudWatch advanced features
- Systems Manager automation
- SNS/EventBridge integration
- Log analysis and metrics
- Custom metrics and dashboards

### Implementation Steps:
1. Set up comprehensive monitoring:
```yaml
# CloudWatch Agent Configuration
{
  "metrics": {
    "metrics_collected": {
      "mem": {
        "measurement": ["mem_used_percent", "mem_total", "mem_available"],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": ["disk_used_percent", "disk_free"],
        "ignore_file_system_types": ["sysfs", "devtmpfs"],
        "metrics_collection_interval": 60
      },
      "netstat": {
        "measurement": ["tcp_established", "tcp_time_wait"],
        "metrics_collection_interval": 60
      }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/syslog",
            "log_group_name": "syslog",
            "log_stream_name": "{instance_id}"
          },
          {
            "file_path": "/var/log/auth.log",
            "log_group_name": "auth_log",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  }
}
```

2. Create automated remediation:
```yaml
# Systems Manager Automation Document
---
schemaVersion: '0.3'
description: 'Automated EC2 Recovery'
parameters:
  InstanceId:
    type: String
    description: EC2 instance to recover
mainSteps:
  - name: checkInstance
    action: aws:assertAwsResourceProperty
    inputs:
      Service: ec2
      Api: DescribeInstanceStatus
      InstanceIds:
        - '{{ InstanceId }}'
      PropertySelector: '$.InstanceStatuses[0].InstanceStatus.Status'
      DesiredValues:
        - impaired
  - name: stopInstance
    action: aws:executeAwsApi
    inputs:
      Service: ec2
      Api: StopInstances
      InstanceIds: 
        - '{{ InstanceId }}'
  - name: waitForStop
    action: aws:waitForAwsResourceProperty
    inputs:
      Service: ec2
      Api: DescribeInstances
      InstanceIds:
        - '{{ InstanceId }}'
      PropertySelector: '$.Reservations[0].Instances[0].State.Name'
      DesiredValues:
        - stopped
  - name: startInstance
    action: aws:executeAwsApi
    inputs:
      Service: ec2
      Api: StartInstances
      InstanceIds:
        - '{{ InstanceId }}'
```

3. Set up alerting thresholds:
```yaml
Resources:
  CPUAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: HighCPUUtilization
      ComparisonOperator: GreaterThanThreshold
      EvaluationPeriods: 2
      MetricName: CPUUtilization
      Namespace: AWS/EC2
      Period: 300
      Statistic: Average
      Threshold: 80
      AlarmActions: 
        - !Ref NotificationTopic
      Dimensions:
        - Name: AutoScalingGroupName
          Value: !Ref ASG
```

### Key Concepts Covered:
- Advanced monitoring configuration
- Automated remediation
- Log aggregation and analysis
- Custom metrics and alarms
- Operational automation

## Project 2: High Availability Infrastructure Deployment
### Learning Objectives:
- Auto Scaling configurations
- Load Balancer management
- Multi-AZ deployments
- Backup and recovery
- Infrastructure as Code

### Implementation Steps:
1. Create high availability architecture:
```yaml
# CloudFormation Template
Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: HA-VPC

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: 10.0.1.0/24
      MapPublicIpOnLaunch: true

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: 10.0.2.0/24
      MapPublicIpOnLaunch: true

  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Subnets:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2
      SecurityGroups:
        - !Ref ALBSecurityGroup

  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      VPCZoneIdentifier:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2
      LaunchTemplate:
        LaunchTemplateId: !Ref LaunchTemplate
        Version: !GetAtt LaunchTemplate.LatestVersionNumber
      MinSize: 2
      MaxSize: 6
      DesiredCapacity: 2
      HealthCheckType: ELB
      HealthCheckGracePeriod: 300
      TargetGroupARNs:
        - !Ref ALBTargetGroup
```

2. Implement backup strategy:
```yaml
# AWS Backup configuration
BackupVault:
  Type: AWS::Backup::BackupVault
  Properties:
    BackupVaultName: production-backup-vault

BackupPlan:
  Type: AWS::Backup::BackupPlan
  Properties:
    BackupPlan:
      BackupPlanName: production-backup-plan
      BackupPlanRule:
        - RuleName: DailyBackups
          TargetBackupVault: !Ref BackupVault
          ScheduleExpression: cron(0 5 ? * * *)
          StartWindowMinutes: 60
          CompletionWindowMinutes: 120
          Lifecycle:
            DeleteAfterDays: 30
```

### Key Concepts Covered:
- High availability design
- Auto Scaling management
- Load balancer configuration
- Backup strategies
- Infrastructure deployment

## Project 3: Security and Compliance Implementation
### Learning Objectives:
- AWS Config rules
- Security Hub integration
- GuardDuty monitoring
- IAM best practices
- Compliance reporting

### Implementation Steps:
1. Set up AWS Config rules:
```yaml
Resources:
  ConfigRule:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: required-tags
      Description: Checks if resources have required tags
      Scope:
        ComplianceResourceTypes:
          - AWS::EC2::Instance
          - AWS::RDS::DBInstance
      Source:
        Owner: AWS
        SourceIdentifier: REQUIRED_TAGS
      InputParameters:
        tag1Key: Environment
        tag1Value: Production,Development,Staging
        tag2Key: Owner
        tag2Value: TeamA,TeamB,TeamC

  EncryptionRule:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: encrypted-volumes
      Description: Checks if EBS volumes are encrypted
      Scope:
        ComplianceResourceTypes:
          - AWS::EC2::Volume
      Source:
        Owner: AWS
        SourceIdentifier: ENCRYPTED_VOLUMES
```

2. Implement security monitoring:
```python
# Lambda function for security findings
def process_security_finding(event, context):
    finding = event['detail']
    severity = finding['Severity']['Label']
    
    if severity in ['CRITICAL', 'HIGH']:
        # Create incident in ticket system
        create_incident(finding)
        
        # Send notification
        sns.publish(
            TopicArn=SECURITY_TOPIC_ARN,
            Message=json.dumps({
                'severity': severity,
                'finding': finding['Title'],
                'resource': finding['Resources'][0]['Id']
            })
        )
```

### Key Concepts Covered:
- Security monitoring
- Compliance automation
- Access management
- Security incident response
- Audit logging

## Project 4: Cost Optimization and Resource Management
### Learning Objectives:
- Cost allocation tags
- Resource scheduling
- Reserved Instance management
- Budget monitoring
- Resource cleanup

### Implementation Steps:
1. Implement cost management:
```yaml
# AWS Budgets configuration
Resources:
  CostBudget:
    Type: AWS::Budgets::Budget
    Properties:
      Budget:
        BudgetName: Monthly-Cost-Budget
        BudgetLimit:
          Amount: 1000
          Unit: USD
        TimeUnit: MONTHLY
        BudgetType: COST
        CostFilters:
          TagKeyValue:
            - Key: Environment
              Value: Production
      NotificationsWithSubscribers:
        - Notification:
            NotificationType: ACTUAL
            ComparisonOperator: GREATER_THAN
            Threshold: 80
          Subscribers:
            - SubscriptionType: EMAIL
              Address: team@example.com
```

2. Create resource scheduling:
```python
# Lambda function for resource scheduling
def schedule_resources(event, context):
    # Stop non-production instances outside business hours
    ec2.describe_instances(
        Filters=[
            {'Name': 'tag:Environment', 'Values': ['Development']},
            {'Name': 'instance-state-name', 'Values': ['running']}
        ]
    )
    
    for instance in response['Reservations']:
        ec2.stop_instances(
            InstanceIds=[instance['InstanceId']]
        )
```

### Key Concepts Covered:
- Cost monitoring
- Resource optimization
- Automated scheduling
- Budget management
- Resource lifecycle management

## Additional Tips:
1. Use AWS Organizations for multi-account management
2. Implement proper tagging strategy
3. Use Infrastructure as Code
4. Regular security assessments
5. Monitor and optimize costs
6. Document operational procedures

## Important Considerations:
- Follow AWS Well-Architected Framework
- Implement proper backup strategies
- Use encryption for sensitive data
- Regular security patching
- Monitor compliance requirements
