# Complete WordPress Infrastructure - Deployment Guide

> **Production-ready, highly available WordPress architecture using AWS CloudFormation nested stacks**

## 🚀 Overview

This project automates the deployment of a complete WordPress infrastructure on AWS using nested CloudFormation stacks. The architecture provides high availability, auto-scaling, and enterprise-grade security features.

### ✨ Key Features

- **Network Isolation**: Dedicated VPC with proper subnet segmentation
- **High Availability**: Multi-AZ deployment across all tiers
- **Auto Scaling**: Dynamic scaling based on CPU utilization
- **Enhanced Security**: Encrypted storage, least-privilege IAM, and layered security groups
- **Production Ready**: CloudWatch monitoring, automated backups, and health checks

## 🏗️ Architecture

### Network Isolation
- **VPC CIDR**: `172.16.0.0/16`
- **VPC Name**: `WP-Production-VPC`
- **No conflicts** with existing lab environments

### Enhanced Security
- ✅ **EFS Encryption**: Enabled by default
- ✅ **Database Encryption**: Enabled with Aurora
- ✅ **IAM Roles**: Proper least-privilege access
- ✅ **Security Groups**: Tighter rules, no `0.0.0.0/0` for app tier

### Production Features
- ✅ **CloudWatch Alarms**: Auto scaling based on CPU
- ✅ **EFS Access Points**: Better security and permissions
- ✅ **Backup Policies**: 7-day backup retention
- ✅ **CloudFormation Signals**: Proper instance health validation

## 📁 File Structure

```
wordpress-infrastructure/
├── master-stack.yaml           # Main orchestration template
├── nested-stacks/
│   ├── network-stack.yaml      # VPC, subnets, security groups
│   ├── database-stack.yaml     # Aurora MySQL cluster
│   ├── storage-stack.yaml      # EFS file system
│   └── compute-stack.yaml      # ALB, ASG, Launch Template
└── README.md                   # This file
```

## 🚀 Deployment Steps

### Step 1: Upload Templates to S3

```bash
# Create S3 bucket (replace with your bucket name)
aws s3 mb s3://your-cf-templates-bucket-name

# Upload nested stack templates
aws s3 cp nested-stacks/network-stack.yaml s3://your-cf-templates-bucket-name/nested-stacks/
aws s3 cp nested-stacks/database-stack.yaml s3://your-cf-templates-bucket-name/nested-stacks/
aws s3 cp nested-stacks/storage-stack.yaml s3://your-cf-templates-bucket-name/nested-stacks/
aws s3 cp nested-stacks/compute-stack.yaml s3://your-cf-templates-bucket-name/nested-stacks/
```

### Step 2: Update Master Template

Replace `your-cf-templates-bucket` in the master template with your actual bucket name:

```yaml
Parameters:
  S3BucketName:
    Default: your-actual-bucket-name  # Update this
```

### Step 3: Deploy via Console

1. Go to **CloudFormation Console**
2. **Create Stack** → **With new resources**
3. **Upload** `master-stack.yaml`
4. **Fill in parameters**:
   - `EnvironmentName`: `WP-Production`
   - `S3BucketName`: `your-actual-bucket-name`
   - `DBMasterPassword`: Strong password (8+ chars)
   - `WordPressAdminPassword`: Strong password (8+ chars)
   - `WordPressAdminEmail`: Your email address

### Step 4: Deploy via CLI

```bash
aws cloudformation create-stack \
  --stack-name wordpress-production \
  --template-body file://master-stack.yaml \
  --parameters \
    ParameterKey=S3BucketName,ParameterValue=your-cf-templates-bucket-name \
    ParameterKey=DBMasterPassword,ParameterValue=YourSecurePassword123! \
    ParameterKey=WordPressAdminPassword,ParameterValue=YourWPPassword123! \
    ParameterKey=WordPressAdminEmail,ParameterValue=admin@yourdomain.com \
  --capabilities CAPABILITY_NAMED_IAM
```

## ⏱️ Deployment Timeline

| Stack | Duration | Notes |
|-------|----------|-------|
| Network Stack | ~5 minutes | VPC, subnets, NAT gateways |
| Database Stack | ~10-15 minutes | Aurora takes time |
| Storage Stack | ~3 minutes | EFS creation |
| Compute Stack | ~8-12 minutes | ALB, ASG, instances |
| **Total** | **~25-35 minutes** | Complete deployment |

## 🔧 What Gets Created

### Network Architecture (172.16.0.0/16)
```
├── Public Subnets (172.16.0.0/24, 172.16.1.0/24)
│   ├── Internet Gateway
│   ├── NAT Gateways
│   └── Application Load Balancer
├── App Subnets (172.16.2.0/24, 172.16.3.0/24)
│   ├── EC2 Instances (Auto Scaling)
│   └── EFS Mount Targets
└── Database Subnets (172.16.4.0/24, 172.16.5.0/24)
    └── Aurora MySQL Cluster
```

### Security Groups
- **ALB-SG**: HTTP/HTTPS from internet
- **App-SG**: HTTP from ALB only
- **DB-SG**: MySQL from App-SG only
- **EFS-SG**: NFS from App-SG only

### Auto Scaling Configuration
- **Min**: 2 instances
- **Max**: 4 instances
- **Scale Up**: CPU > 70% for 10 minutes
- **Scale Down**: CPU < 25% for 10 minutes

## 🎯 Testing Your Deployment

### 1. Get WordPress URL

```bash
# Get the load balancer DNS name
aws cloudformation describe-stacks \
  --stack-name wordpress-production \
  --query 'Stacks[0].Outputs[?OutputKey==`WordPressURL`].OutputValue' \
  --output text
```

### 2. Access WordPress

- **Frontend**: `http://your-alb-dns-name`
- **Admin**: `http://your-alb-dns-name/wp-admin`
- **Login**: `wpadmin` / `your-password`

### 3. Test High Availability

```bash
# Terminate an instance - ASG will replace it
aws ec2 terminate-instances --instance-ids i-xxxxxxxxx

# Upload a file to WordPress, then terminate instance
# File should still be accessible (EFS working)
```

## 💰 Cost Optimization

### Development Environment
```yaml
# Reduce costs for dev/test
InstanceType: t3.small          # Instead of t3.medium
AuroraInstanceClass: db.t3.small # Instead of db.t3.medium
MinSize: 1                      # Instead of 2
MaxSize: 2                      # Instead of 4
```

### Production Environment
```yaml
# For higher traffic
InstanceType: t3.large
AuroraInstanceClass: db.r5.large
MinSize: 3
MaxSize: 10
```

## 🔒 Security Enhancements

### SSL/TLS (Recommended)
```yaml
# Add to ALB Listener
Certificate: arn:aws:acm:region:account:certificate/cert-id
Protocol: HTTPS
Port: 443
```

### WAF (Recommended)
```yaml
# Add Web Application Firewall
WebACL:
  Type: AWS::WAFv2::WebACL
  Rules:
    - AWS Managed Rules
    - Rate limiting
    - IP blocking
```

## 🚨 Troubleshooting

### Common Issues

**1. Stack Creation Fails**
- Check S3 bucket permissions
- Verify template URLs are accessible
- Ensure IAM permissions for nested stacks

**2. Instances Unhealthy**
- Check security group rules
- Verify EFS mount is working
- Check CloudWatch logs in `/var/log/messages`

**3. WordPress Installation Fails**
- Verify database connectivity
- Check Aurora security groups
- Ensure EFS is mounted properly

**4. Auto Scaling Not Working**
- Check CloudWatch alarms
- Verify IAM permissions
- Review ASG health check settings

### Debug Commands
```bash
# SSH into instance (if you add a key pair)
sudo tail -f /var/log/messages
sudo mount | grep efs
sudo netstat -tuln | grep 3306
sudo systemctl status httpd
```

## 📈 Monitoring & Maintenance

### CloudWatch Dashboards
- ALB request count and latency
- EC2 CPU and memory utilization
- Aurora connection count and CPU
- EFS throughput and IOPS

### Backup Strategy
- **Aurora**: Automated 7-day backup retention
- **EFS**: Consider AWS Backup for point-in-time recovery
- **WordPress**: Regular database exports via WP-CLI

### Updates
```bash
# Update stack with new parameters
aws cloudformation update-stack \
  --stack-name wordpress-production \
  --template-body file://master-stack.yaml \
  --parameters ParameterKey=InstanceType,ParameterValue=t3.large
```

## 🎯 Key Advantages Over Manual Setup

| Feature | Manual Setup | Automated Stack |
|---------|--------------|-----------------|
| **Consistency** | ❌ Prone to errors | ✅ Identical every time |
| **Speed** | ⏱️ 3+ hours | ⚡ 30 minutes |
| **Rollback** | ❌ Manual process | ✅ Easy revert |
| **Scaling** | ❌ Reconfigure manually | ✅ Update parameters |
| **Documentation** | ❌ Tribal knowledge | ✅ Infrastructure as code |
| **Testing** | ❌ Hard to reproduce | ✅ Deploy dev/staging easily |

## 🔥 Next Level Enhancements

### CI/CD Pipeline
```yaml
# Add to compute stack
CodePipeline:
  - Source: GitHub
  - Build: CodeBuild
  - Deploy: CodeDeploy to ASG
```

### Multi-Region Deployment
```yaml
# Cross-region replication
Aurora: Global Database
EFS: Backup to different region
Route53: Health checks and failover
```

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add some amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 📞 Support

If you encounter any issues or have questions:

1. Check the [Troubleshooting](#-troubleshooting) section
2. Review AWS CloudFormation logs
3. Open an issue in this repository

---

> **Note**: This automated approach transforms manual infrastructure deployment into a production-ready, enterprise-grade WordPress infrastructure that can be deployed in under 30 minutes!