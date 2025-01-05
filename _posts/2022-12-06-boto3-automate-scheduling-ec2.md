---
title: 'Boto3 - Automate Scheduling of EC2 Instances: Using Lambda and Cron'
tags:
- boto3
- lambda
- aws
- EventBridge
- python
---

![image info](assets/images/boto-ec2/boto3_lambda.png){: width="250" }

Managing the aws bill and availability of your ec2 instances can be challenging if they run all the time without any need. For example, many teams / companies often have development instances that are not used after working hours and weekends. So, by shutting these down when not in use, you could save a significant amount on your AWS bill.

In our case, we tag all development instances (or any other resources that don’t need to run outside working hours) with `env: dev`. This makes it easy to filter and manage these instances separately from production or other critical workloads.

![image info](assets/images/boto-ec2/ec2-screen.png){: width="550" }

In this post, we’ll walk you through how to automate the process of stopping and starting these tagged instances using two approaches:

1. Using **AWS Lambda**.
2. Using an **on-premises machine with Cron**.

Both methods will utilize **Boto3**, the AWS SDK for Python.

---

### Prerequisites

Before starting, ensure you:

- Have Python installed (for on-prem).
- Install Boto3 using:
    
    ```
    pip install boto3
    ```
    
- Set up AWS credentials using `aws configure` or environment variables.
- Grant your IAM role or user sufficient permissions:
    - `ec2:DescribeInstances`
    - `ec2:StopInstances`
    - `ec2:StartInstances`
    
![image info](assets/images/boto-ec2/iam.png){: width="450" }

---

### **Approach 1: Using AWS Lambda to Schedule EC2 Instances**

AWS Lambda, combined with Amazon EventBridge (CloudWatch Events), can automate the start and stop of EC2 instances.

### 1.1: Lambda Script to Stop EC2 Instances

This Lambda script stops EC2 instances with the tag `env=dev` in the regions ex. `us-east-1`, `us-east-2` , `us-west-1` and `us-west-2` if they are running.

```
import boto3
import json

def lambda_handler(event, context):
    ### HERE, set your regions in this list
    regions = ['us-east-1', 'us-east-2','us-west-1', 'us-west-2']
    for region_name in regions:
        print(f"Checking region: {region_name}")
        ec2_resource = boto3.resource('ec2', region_name=region_name)
				
     ### Filter only running instances with Tag env=dev
        filters = [
            {'Name': 'instance-state-name', 'Values': ['running']},  
            {'Name': 'tag:env', 'Values': ['dev']}                  
        ]

        instances = list(ec2_resource.instances.filter(Filters=filters))

        if instances:
            print(f"Instances in region {region_name}:")
            for instance in instances:
                print(f"Stopping Instance ID: {instance.id} State: {instance.state['Name']}")
                instance.stop()  
        else:
            print(f"No running 'dev' EC2 instances in region {region_name}")

    return {
        'statusCode': 200,
        'body': json.dumps('EC2 instances stopped successfully!')
    }
```

### 1.2: Lambda Script to Start EC2 Instances

This Lambda script will starts EC2 instances with the tag `env=dev` if they are stopped.

```
import boto3
import json

def lambda_handler(event, context):
    regions = ['us-east-1', 'us-east-2','us-west-1', 'us-west-2']
    for region_name in regions:
        print(f"Checking region: {region_name}")
        ec2_resource = boto3.resource('ec2', region_name=region_name)
        filters = [
            {'Name': 'instance-state-name', 'Values': ['stopped']},  
            {'Name': 'tag:env', 'Values': ['dev']}                  
        ]

        instances = list(ec2_resource.instances.filter(Filters=filters))
        if instances:
            print(f"Instances in region {region_name}:")
            for instance in instances:
                print(f"Starting Instance ID: {instance.id}")
                instance.start()  
        else:
            print(f"No stopped 'dev' EC2 instances in region {region_name}")

    return {
        'statusCode': 200,
        'body': json.dumps('EC2 instances started successfully!')
    }
```

### Timeouts Troubleshooting Note:
When you run  AWS Lambda you might face timeout error.
![image info](assets/images/boto-ec2/timeout-lambda.png){: width="550" }


So, it’s important to configure the timeout to ensure the function has enough time (ex. 30 seconds) to complete its task. The default timeout is **3 seconds**, which might not be sufficient for scripts that need to iterate over multiple regions/instances.

### 1.3: Scheduling with EventBridge

1. Go to the **EventBridge Console** in AWS.
2. Create a new rule:
    -  Name your rule -  ex: `stop-ec2-dev-instances`
    - And schedule expression:
        - Stop at 6 PM: `cron(0 18 ? * MON-FRI *)`
        - Start at 6 AM: `cron(0 6 ? * MON-FRI *)`
3. Set the target as your Lambda function.

---

### **Approach 2: Using Cron on an On-Premises Machine**

If you prefer to manage this locally, use the following scripts and schedule them with `cron`.

### 2.1: Stopping EC2 Instances Script

```
import boto3

def list_used_ec2_instances():
    regions = ['us-east-1', 'us-east-2','us-west-1', 'us-west-2']
    for region_name in regions:
        print(f"Checking region: {region_name}")
        ec2_resource = boto3.resource('ec2', region_name=region_name)
        filters = [
            {'Name': 'instance-state-name', 'Values': ['running']},
            {'Name': 'tag:env', 'Values': ['dev']}
        ]

        instances = list(ec2_resource.instances.filter(Filters=filters))

        if instances:
            print(f"Instances in region {region_name}:")
            for instance in instances:
                print(f"Stopping Instance ID: {instance.id}")
                instance.stop()
        else:
            print(f"No running 'dev' EC2 instances in region {region_name}")

list_used_ec2_instances()
```
![image info](assets/images/boto-ec2/stop-ec2.png){: width="550" }

### 2.2: Starting EC2 Instances Script

```
import boto3

def list_used_ec2_instances():
    regions = ['us-east-1', 'us-east-2','us-west-1', 'us-west-2']
    for region_name in regions:
        print(f"Checking region: {region_name}")
        ec2_resource = boto3.resource('ec2', region_name=region_name)

        filters = [
            {'Name': 'instance-state-name', 'Values': ['stopped']},
            {'Name': 'tag:env', 'Values': ['dev']}
        ]

        instances = list(ec2_resource.instances.filter(Filters=filters))

        if instances:
            print(f"Instances in region {region_name}:")
            for instance in instances:
                print(f"Starting Instance ID: {instance.id}")
                instance.start()
        else:
            print(f"No stopped 'dev' EC2 instances in region {region_name}")

list_used_ec2_instances()
```
![image info](assets/images/boto-ec2/start-ec2.png){: width="550" }

### 2.3: Schedule with Cron

1. Open the crontab editor:
    
    ```
    crontab -e
    ```
    
2. Add the following entries:
    
    ```
    0 18 * * 1-5 /usr/bin/python3 /scripts/stop_ec2.py
    0 6 * * 1-5 /usr/bin/python3 /scripts/start_ec2.py
    ```
    
    - **0 18**: Runs the stop script at 6 PM, Monday to Friday.
    - **0 6**: Runs the start script at 6 AM, Monday to Friday.
