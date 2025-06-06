Complete WordPress Infrastructure - Deployment Guide
Network Isolation

VPC CIDR: 172.16.0.0/16
VPC Name: WP-Production-VPC

Enhanced Security

EFS Encryption: Enabled by default
Database Encryption: Enabled with Aurora
IAM Roles: Proper least-privilege access
Security Groups: Tighter rules, no 0.0.0.0/0 for app tier

Production Features

CloudWatch Alarms: Auto scaling based on CPU
EFS Access Points: Better security and permissions
Backup Policies: 7-day backup retention
CloudFormation Signals: Proper instance health validation

📁 File Structure
wordpress-infrastructure/
├── master-stack.yaml           # Main orchestration template
├── nested-stacks/
│   ├── network-stack.yaml      # VPC, subnets, security groups
│   ├── database-stack.yaml     # Aurora MySQL cluster
│   ├── storage-stack.yaml      # EFS file system
│   └── compute-stack.yaml      # ALB, ASG, Launch Template
└── README.md                   # This file
🚀 Deployment Steps
Step 1: Upload Templates to S3
bash# Create S3 bucket (replace with your bucket name)
aws s3 mb s3://your-cf-templates-bucket-name

# Upload nested stack templates
aws s3 cp network-stack.yaml s3://your-cf-templates-bucket-name/
aws s3 cp database-stack.yaml s3://your-cf-templates-bucket-name/
aws s3 cp storage-stack.yaml s3://your-cf-templates-bucket-name/
aws s3 cp compute-stack.yaml s3://your-cf-templates-bucket-name/
Step 2: Update Master Template
Replace your-s3-bucket in the master template with your actual bucket name:
yamlTemplateURL: !Sub 'https://your-actual-bucket-name.s3.amazonaws.com/network-stack.yaml'
Step 3: Deploy via Console

Go to CloudFormation Console
Create Stack → With new resources
Upload master-stack.yaml
Fill in parameters:

EnvironmentName: WP-Production
DBMasterPassword: Strong password (8+ chars)
WordPressAdminPassword: Strong password (8+ chars)
WordPressAdminEmail: Your email address



Step 4: Deploy via CLI
bashaws cloudformation create-stack \
  --stack-name wordpress-production \
  --template-body file://master-stack.yaml \
  --parameters \
    ParameterKey=DBMasterPassword,ParameterValue=YourSecurePassword123! \
    ParameterKey=WordPressAdminPassword,ParameterValue=YourWPPassword123! \
    ParameterKey=WordPressAdminEmail,ParameterValue=admin@yourdomain.com \
  --capabilities CAPABILITY_NAMED_IAM
⏱️ Deployment Timeline

Network Stack: ~5 minutes
Database Stack: ~10-15 minutes (Aurora takes time)
Storage Stack: ~3 minutes
Compute Stack: ~8-12 minutes
Total: ~25-35 minutes

🔧 What Gets Created
Network (172.16.0.0/16)
├── Public Subnets (172.16.0.0/24, 172.16.1.0/24)
│   ├── Internet Gateway
│   ├── NAT Gateways
│   └── Application Load Balancer
├── App Subnets (172.16.2.0/24, 172.16.3.0/24)
│   ├── EC2 Instances (Auto Scaling)
│   └── EFS Mount Targets
└── Database Subnets (172.16.4.0/24, 172.16.5.0/24)
    └── Aurora MySQL Cluster
Security Groups

ALB-SG: HTTP/HTTPS from internet
App-SG: HTTP from ALB only
DB-SG: MySQL from App-SG only
EFS-SG: NFS from App-SG only

Auto Scaling

Min: 2 instances
Max: 4 instances
Scale Up: CPU > 70% for 10 minutes
Scale Down: CPU < 25% for 10 minutes

🎯 Testing Your Deployment
1. Get WordPress URL
bash# Get the load balancer DNS name
aws cloudformation describe-stacks \
  --stack-name wordpress-production \
  --query 'Stacks[0].Outputs[?OutputKey==`WordPressURL`].OutputValue' \
  --output text
2. Access WordPress

Frontend: http://your-alb-dns-name
Admin: http://your-alb-dns-name/wp-admin
Login: wpadmin / your-password

3. Test High Availability
bash# Terminate an instance - ASG will replace it
aws ec2 terminate-instances --instance-ids i-xxxxxxxxx

# Upload a file to WordPress, then terminate instance
# File should still be accessible (EFS working)
💰 Cost Optimization
Development Environment
yaml# Reduce costs for dev/test
InstanceType: t3.small          # Instead of t3.medium
AuroraInstanceClass: db.t3.small # Instead of db.t3.medium
MinSize: 1                      # Instead of 2
MaxSize: 2                      # Instead of 4
Production Environment
yaml# For higher traffic
InstanceType: t3.large
AuroraInstanceClass: db.r5.large
MinSize: 3
MaxSize: 10
🔒 Security Enhancements
SSL/TLS (Recommended)
yaml# Add to ALB Listener
Certificate: arn:aws:acm:region:account:certificate/cert-id
Protocol: HTTPS
Port: 443
WAF (Recommended)
yaml# Add Web Application Firewall
WebACL:
  Type: AWS::WAFv2::WebACL
  Rules:
    - AWS Managed Rules
    - Rate limiting
    - IP blocking
🚨 Troubleshooting
Common Issues
1. Stack Creation Fails

Check S3 bucket permissions
Verify template URLs are accessible
Ensure IAM permissions for nested stacks

2. Instances Unhealthy

Check security group rules
Verify EFS mount is working
Check CloudWatch logs in /var/log/messages

3. WordPress Installation Fails

Verify database connectivity
Check Aurora security groups
Ensure EFS is mounted properly

4. Auto Scaling Not Working

Check CloudWatch alarms
Verify IAM permissions
Review ASG health check settings

Debug Commands
bash# SSH into instance (if you add a key pair)
sudo tail -f /var/log/messages
sudo mount | grep efs
sudo netstat -tuln | grep 3306
sudo systemctl status httpd
📈 Monitoring & Maintenance
CloudWatch Dashboards

ALB request count and latency
EC2 CPU and memory utilization
Aurora connection count and CPU
EFS throughput and IOPS

Backup Strategy

Aurora: Automated 7-day backup retention
EFS: Consider AWS Backup for point-in-time recovery
WordPress: Regular database exports via WP-CLI

Updates
bash# Update stack with new parameters
aws cloudformation update-stack \
  --stack-name wordpress-production \
  --template-body file://master-stack.yaml \
  --parameters ParameterKey=InstanceType,ParameterValue=t3.large
🎯 Key Advantages Over Manual Setup

Consistency: Identical environments every time
Speed: 30 minutes vs 3+ hours manual
Rollback: Easy to revert changes
Scaling: Modify parameters and update
Documentation: Infrastructure as code
Testing: Deploy dev/staging environments easily

🔥 Next Level Enhancements
CI/CD Pipeline
yaml# Add to compute stack
CodePipeline:
  - Source: GitHub
  - Build: CodeBuild
  - Deploy: CodeDeploy to ASG
Multi-Region Deployment
yaml# Cross-region replication
Aurora: Global Database
EFS: Backup to different region
Route53: Health checks and failover
This automated approach transforms your manual lab into a production-ready, enterprise-grade WordPress infrastructure that can be deployed in under 30 minutes!