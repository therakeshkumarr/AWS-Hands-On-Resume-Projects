# AWS-Hands-On-Resume-Projects
This is Complete Console-Based Guide. This guide is ready-to-use, beginner-friendly, and includes full instructions for building three AWS projects with testing and monitoring.

# PROJECT 1: Highly Available Web Application (EC2 + ALB + ASG)
Goal: Deploy a web app that automatically scales and is highly available.

**Steps:**
1. **Create VPC**

o   VPC → Your VPCs → Create VPC

o   Name: ha-vpc | IPv4 CIDR: 10.0.0.0/16 | Enable DNS hostnames

2. **Create Subnets**

o   Public subnets (for ALB): 10.0.1.0/24 → AZ1 | 10.0.2.0/24 → AZ2 | Enable Auto-assign public IPv4

o   Private subnets (for EC2): 10.0.3.0/24 → AZ1 | 10.0.4.0/24 → AZ2

3. **Internet Gateway**

o   VPC → Internet Gateways → Create → Attach to ha-vpc

4. **NAT Gateway**

o   NAT Gateways → Create → Subnet: Public 1 → Allocate Elastic IP

5. **Route Tables**

o   Public → route 0.0.0.0/0 → IGW | Associate public subnets

o   Private → route 0.0.0.0/0 → NAT | Associate private subnets

6. **Security Groups**

o   ALB SG: inbound HTTP 80 from anywhere

o   EC2 SG: inbound HTTP 80 from ALB SG only

7. **Launch EC2**

o   EC2 → Launch Instance → Amazon Linux 2 → t3.micro

o   Security Group: EC2 SG

o   User data script:

    yum install -y httpd
systemctl start httpd
echo "Healthy from $(hostname)" > /var/www/html/index.html

8. **Create Auto Scaling Group**

o   EC2 → Auto Scaling Groups → Launch template → Private subnets → Desired 2, Min 2, Max 4

o   Attach ALB target group

9. **Create ALB**

o   ALB → Internet-facing → Public subnets → Security Group: ALB SG

o   Target group → attach ASG instances

10. **Validation**

·       Open ALB DNS → Refresh → See page load from different instances


# PROJECT 2: Serverless REST API (API Gateway + Lambda + DynamoDB)
Goal: Build a serverless API performing CRUD operations.

Steps:
1. **Create DynamoDB Table**

o   Table name: users | Partition key: id (String) | On-demand capacity

2.        Create IAM Role for Lambda

o   IAM → Roles → Create Role → Trusted entity: Lambda

o   Attach permissions: AWSLambdaBasicExecutionRole + DynamoDB read/write policy

3. **Create Lambda Function**

o   Lambda → Create → Python → Attach IAM role

o   Add code: ```python import json import boto3 from boto3.dynamodb.conditions import Key

dynamodb = boto3.resource(‘dynamodb’) table = dynamodb.Table(‘users’)

def lambda_handler(event, context): method = event[‘httpMethod’] if method == ‘POST’: body = json.loads(event[‘body’]) table.put_item(Item=body) return {‘statusCode’: 200, ‘body’: json.dumps(‘User created’)}

elif method == 'GET':
        user_id = event['queryStringParameters']['id']
        response = table.get_item(Key={'id': user_id})
        item = response.get('Item', {})
        return {'statusCode': 200, 'body': json.dumps(item)}

elif method == 'PUT':
        body = json.loads(event['body'])
        user_id = body['id']
    table.update_item(
        Key={'id': user_id},
        UpdateExpression="set info=:i",
        ExpressionAttributeValues={':i': body['info']}
        )
        return {'statusCode': 200, 'body': json.dumps('User updated')}

elif method == 'DELETE':
        user_id = event['queryStringParameters']['id']
    table.delete_item(Key={'id': user_id})
        return {'statusCode': 200, 'body': json.dumps('User deleted')}

else:
        return {'statusCode': 400, 'body': json.dumps('Unsupported method')}


4. **Create API Gateway**
   - REST API → Name: `users-api` → Resource `/users` → Methods: GET, POST, PUT, DELETE → Integrate with Lambda
   - Enable CORS → Deploy → Stage: `dev`

5. **Test API**
- Invoke URL: `https://abcd1234.execute-api.us-east-1.amazonaws.com/dev`
- Browser (GET): `https://abcd1234.execute-api.us-east-1.amazonaws.com/dev/users?id=123`
- 
- curl POST:
```bash
curl -X POST https://abcd1234.execute-api.us-east-1.amazonaws.com/dev/users \
-H "Content-Type: application/json" \
-d '{"id":"123","name":"Alice","info":"Tester"}'

- curl GET:
curl -X GET "https://abcd1234.execute-api.us-east-1.amazonaws.com/dev/users?id=123"

- curl PUT:
curl -X PUT https://abcd1234.execute-api.us-east-1.amazonaws.com/dev/users \
-H "Content-Type: application/json" \
-d '{"id":"123","info":"Updated info"}'

- curl DELETE:
curl -X DELETE "https://abcd1234.execute-api.us-east-1.amazonaws.com/dev/users?id=123"

Validate POST → GET → PUT → GET → DELETE → GET sequence
'''

# PROJECT 3: Centralized Monitoring & Alerting (CloudWatch + SNS)
Goal: Detect failures and alert proactively.

Steps:
1.    Install CloudWatch Agent on EC2

# Connect to EC2
ssh -i /path/to/key.pem ec2-user@<EC2_PUBLIC_IP>

# Update packages
sudo yum update -y

# Download agent
wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm

# Install agent
sudo rpm -U ./amazon-cloudwatch-agent.rpm

2.    Configure Agent

sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard

·       Choose EC2

·       OS: Amazon Linux

·       Metrics: CPU, memory, disk, network (as needed)

·       Logs: Yes → e.g., /var/log/messages, /var/log/httpd/access_log

·       CloudWatch Logs group: /ec2/app-logs

·       IAM Role: CloudWatchAgentServerPolicy

3.    Start Agent

sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-a fetch-config \
-m ec2 \
-c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json \
-s

4.    Verify Logs in CloudWatch

·       Go to CloudWatch → Logs → Log Groups → /ec2/app-logs

·       Check log streams → see EC2 logs

5.    Create Alarms and SNS Notifications

·       CloudWatch → Alarms → Create alarm (CPU >70%, ALB 5xx)

·       SNS → Create topic alerts → Subscribe email

·       Link alarms to SNS → Receive notifications

6.    Validation

·       Stress EC2 → CPU alarm triggers → email received


Interview Tips: - Explain service purpose, failure handling, security, cost, and scaling

