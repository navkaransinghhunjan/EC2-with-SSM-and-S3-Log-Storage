# EC2 with SSM and S3 Log Storage

This guide walks through setting up an Amazon EC2 instance that integrates with AWS Systems Manager (SSM) for management and sends its logs to an Amazon S3 bucket. The setup uses minimal permissions to ensure security.

## Prerequisites

* AWS CLI installed and configured with appropriate permissions.
* An S3 bucket in the same region for storing logs.

## Steps
### 1. Create an IAM Role with Minimum Permissions

The EC2 instance requires an IAM role with policies that enable it to communicate with SSM and upload logs to S3.

* Go to IAM Console.
* Create a new role for EC2 with the following permissions:

### IAM Policies for Minimum Permissions

1. SSM Managed Policy:

  * Attach the managed policy AmazonSSMManagedInstanceCore for SSM access.

2. Custom S3 Policy for Log Upload:

  * Create a custom policy to give EC2 minimal S3 write access to a specific bucket:
```
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "s3:PutObject",
            "s3:PutObjectAcl"
          ],
          "Resource": "arn:aws:s3:::your-log-bucket-name/*"
        }
      ]
    }
```
 Replace your-log-bucket-name with your bucket name.

3. Attach Policies:

  *  Attach the AmazonSSMManagedInstanceCore policy and the custom S3 policy to the IAM role.
  * Name the role, e.g., EC2_SSM_MinPerms_Role.

### 2. Configure the S3 Bucket

1. Create an S3 Bucket (if not already created):
```
aws s3api create-bucket --bucket your-log-bucket-name --region ap-south-1 --create-bucket-configuration LocationConstraint=ap-south-1
```
2. Enable Bucket Policy for Security (optional):

* To restrict access to only the EC2 IAM role, you can add a bucket policy as follows:
```
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
            "AWS": "arn:aws:iam::YOUR_ACCOUNT_ID:role/EC2_SSM_MinPerms_Role"
          },
          "Action": "s3:PutObject",
          "Resource": "arn:aws:s3:::your-log-bucket-name/*"
        }
      ]
    }
```
Replace YOUR_ACCOUNT_ID and your-log-bucket-name accordingly.

3. Create a CloudWatch Agent Configuration

Save the following JSON configuration file (cloudwatch-config.json) to collect and store log files.
```
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/messages",
            "log_group_name": "ec2-instance-logs",
            "log_stream_name": "{instance_id}",
            "retention_in_days": 7
          }
        ]
      }
    },
    "log_stream_name": "ec2-logs"
  }
}
```
### 4. Create an EC2 Instance with User Data Script

Use the following steps to create the EC2 instance with a user data script to configure SSM and S3 logging.

1. Create User Data Script: Create a script user-data.sh:
```
#!/bin/bash
# Install SSM Agent
sudo yum install -y amazon-ssm-agent
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent

# Install CloudWatch Agent
sudo yum install -y amazon-cloudwatch-agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/etc/cloudwatch-config.json -s

# Send logs to S3 (replace with your bucket name)
echo "*/5 * * * * aws s3 cp /var/log/messages s3://your-log-bucket-name/ec2-logs/$(hostname)-$(date +\%Y-\%m-\%d).log" | sudo tee -a /etc/crontab
```
2. Launch the EC2 Instance:
```
    aws ec2 run-instances \
        --image-id ami-12345678 \ # replace with a valid AMI ID for your region
        --instance-type t2.micro \
        --iam-instance-profile Name=EC2_SSM_MinPerms_Role \
        --user-data file://user-data.sh \
        --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=EC2_with_SSM_S3_Logs}]'
```
Replace ami-12345678 with the desired AMI ID, and your-log-bucket-name with your actual bucket name.

### 5. Verification

1. SSM Access: Check if the instance appears in the Systems Manager Console under Fleet Manager.
2. Logs in S3: After a few minutes, verify that logs are uploaded to the S3 bucket as per the cron job.
