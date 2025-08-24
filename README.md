# CloudLaunch

CloudLaunch is a lightweight platform that demonstrates AWS core services implementation including S3 static website hosting, IAM access controls, and VPC network design. This project showcases fundamental AWS concepts through a practical deployment scenario using the Omnifood website template.

## Code Attribution
The frontend code for this project is "Omnifood." It is a food delivery website template sourced from a publicly available GitHub repository. The website content and branding remain unchanged as the assignment focus is on demonstrating AWS cloud infrastructure skills rather than frontend development.

**Project Name**: Omnifood

**Original Repository**: (https://github.com/MettaSurendhar/Omnifood)

*Note: The website displays "Omnifood" content while being hosted through the CloudLaunch infrastructure setup.*

## Live Links
- **S3 Static Website**: http://cloudlaunch-site-buckett.s3-website-us-east-1.amazonaws.com
- **CloudFront Distribution (HTTPS)**: https://dna7ev6bthlku.cloudfront.net

## Access Credentials
- **Console URL**: https://adewuumii.signin.aws.amazon.com/console
- **Username**: cloudlaunch-user
- **Initial Password**: *1xQ9%SW (set to be changed on first login)

## Visual Documentation
Complete implementation screenshots are available in the `screenshots/` folder:
- `s3-buckets-overview.png` - All three S3 buckets configuration
- `website-live-s3.png` - Omnifood website running on http
- `website-live-cloudfront.png` - Omnifood website running on HTTPS
- `vpc-architecture.png` - Complete VPC network diagram
- `security-groups.png` - Security group rules and configurations
- `cloudfront-distribution.png` - CloudFront distribution details

---

# Task 1: Static Website Hosting Using S3 + IAM

## Step 1: Creating the S3 Buckets

I started by creating three S3 buckets, each with different access configurations to demonstrate various AWS S3 security patterns.

### 1.1 Creating the Public Website Bucket (cloudlaunch-site-buckett)

First, I created the bucket that would host my static website:

1. **Created the bucket**: Named it `cloudlaunch-site-buckett`
2. **Enabled public access**: Unchecked "Block all public access" to allow public website hosting
3. **Enabled static website hosting**:
   - Went to Properties → Static website hosting
   - Set index document to `index.html`
   - Got the bucket website endpoint: `http://cloudlaunch-site-buckett.s3-website-us-east-1.amazonaws.com`

4. **Uploaded website files**: Uploaded all Omnifood website files and folders to the bucket
5. **Made objects publicly accessible**:
   - Went to Permissions → Object Ownership → Enabled ACLs
   - Selected all objects → Actions → "Make public using ACLs"

At this point, I could access the website via the S3 endpoint, but it was HTTP only.

### 1.2 Setting Up CloudFront Distribution 

To add HTTPS support and global caching, I configured CloudFront:

1. **Created Origin Access Control (OAC)**:
   - This allows CloudFront to securely access the S3 bucket while blocking direct public access

2. **Created CloudFront Distribution**:
   - Origin: My S3 bucket
   - Origin Access: Used the OAC I created
   - Viewer Protocol Policy: "Redirect HTTP to HTTPS"
   - Default Root Object: `index.html`
   - Disabled WAF to stay within free tier

3. **Updated S3 Bucket Policy**:
   After creating the distribution, I copied the generated bucket policy and applied it to my S3 bucket:

```json
{
  "Version": "2008-10-17",
  "Id": "PolicyForCloudFrontPrivateContent",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::cloudlaunch-site-buckett/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::686255950557:distribution/E1AR2G4HBC32Q2"
        }
      }
    }
  ]
}
```

This policy ensures only CloudFront can access the bucket, not direct public access.

4. **Handled Cache Invalidation**:
   When I made changes to files, I had to create invalidations in CloudFront:
   - Went to CloudFront → Distributions → Invalidations
   - Created invalidation with object path: `/*`
   - This forces CloudFront to fetch the latest version from S3

### 1.3 Creating Private Buckets

**Private Bucket (cloudlaunch-private-buckett)**:
- Left "Block all public access" enabled
- Disabled ACLs
- This bucket will only be accessible via IAM permissions

**Visible-Only Bucket (cloudlaunch-visible-only-buckett)**:
- Left "Block all public access" enabled
- Disabled ACLs
- Uploaded all Omnifood website files and folders to the bucket as test files
- This demonstrates list-only permissions (user can see the bucket but not access contents)

## Step 2: IAM User and Policy Configuration

### 2.1 Creating the IAM User

1. **Created IAM User**:
   - Username: `cloudlaunch-user`
   - Enabled AWS Management Console access
   - Set custom password: `*1xQ9%SW`
   - Enabled "User must create a new password at next sign-in"

2. **Generated Console URL**: https://adewuumii.signin.aws.amazon.com/console

### 2.2 Creating Custom IAM Policy

I created a custom policy that implements the principle of least privilege:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListAllMyBuckets",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::cloudlaunch-site-buckett",
        "arn:aws:s3:::cloudlaunch-private-buckett",
        "arn:aws:s3:::cloudlaunch-visible-only-buckett"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::cloudlaunch-site-buckett/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::cloudlaunch-private-buckett/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeVpcs",
        "ec2:DescribeSubnets",
        "ec2:DescribeRouteTables",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeInternetGateways",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeNatGateways",
        "ec2:DescribeVpcEndpoints",
        "ec2:DescribeNetworkAcls"
      ],
      "Resource": "*"
    }
  ]
}
```

**Policy Breakdown**:
- **Statement 1**: Allows listing all S3 buckets (required for S3 console navigation)
- **Statement 2**: Allows listing contents of the three specific buckets
- **Statement 3**: Allows reading objects from the site bucket only
- **Statement 4**: Allows reading and writing to the private bucket only (no delete permissions)
- **Statement 5**: Provides read-only access to VPC components for Task 2

### 2.3 Testing IAM Permissions

I logged into the IAM user console and verified:
- ✅ Can see all three buckets in S3 console
- ✅ Can read objects from cloudlaunch-site-buckett
- ✅ Can upload and download from cloudlaunch-private-buckett
- ✅ Can list cloudlaunch-visible-only-buckett but cannot access its objects
- ✅ Can view VPC components in EC2 console

---

# Task 2: VPC Design for CloudLaunch Environment

## Step 1: Creating the VPC

I started by creating a VPC that would serve as the foundation for a secure, scalable network environment.

### 1.1 VPC Configuration
1. **Created VPC**:
   - Name: `cloudlaunch-vpc`
   - IPv4 CIDR block: `10.0.0.0/16`
   - Tenancy: Default
   - Used the "VPC only" option 

This CIDR block provides 65,536 IP addresses, allowing for significant future expansion.

## Step 2: Creating Subnets

I designed a three-tier architecture with subnets in different availability zones for high availability:

### 2.1 Public Subnet
```
Name: cloudlaunch-public-subnet
CIDR: 10.0.1.0/24
Availability Zone: us-east-1a
Purpose: Load balancers and public-facing services
```

### 2.2 Application Subnet (Private)
```
Name: cloudlaunch-app-subnet
CIDR: 10.0.2.0/24
Availability Zone: us-east-1b
Purpose: Application servers
```

### 2.3 Database Subnet (Private)
```
Name: cloudlaunch-db-subnet
CIDR: 10.0.3.0/28
Availability Zone: us-east-1c
Purpose: Database services (only 16 IPs needed)
```

**Note**: a /28 was used for the database subnet since databases typically need fewer instances.

## Step 3: Internet Gateway Setup

### 3.1 Creating and Attaching Internet Gateway
1. **Created Internet Gateway**:
   - Name: `cloudlaunch-igw`
2. **Attached to VPC**:
   - Selected the gateway → Actions → Attach to VPC
   - Selected `cloudlaunch-vpc`

This gateway enables internet access for resources in public subnets.

## Step 4: Route Table Configuration

### 4.1 Public Route Table
1. **Created route table**: `cloudlaunch-public-rt`
2. **Added internet route**:
   - Destination: `0.0.0.0/0`
   - Target: `cloudlaunch-igw`
3. **Associated with public subnet**:
   - Subnet Associations tab → Edit → Selected `cloudlaunch-public-subnet`

### 4.2 Private Route Tables
I created separate route tables for each private subnet to maintain isolation:

**Application Route Table** (`cloudlaunch-app-rt`):
- No internet route (0.0.0.0/0)
- Only local VPC traffic (10.0.0.0/16)
- Associated with `cloudlaunch-app-subnet`

**Database Route Table** (`cloudlaunch-db-rt`):
- No internet route (0.0.0.0/0)
- Only local VPC traffic (10.0.0.0/16)
- Associated with `cloudlaunch-db-subnet`

This ensures private subnets cannot access the internet directly, following security best practices.

## Step 5: Security Group Configuration

### 5.1 Application Security Group
```
Name: cloudlaunch-app-sg
Description: Security group for application servers
Inbound Rules:
- Type: HTTP (80)
- Source: 10.0.0.0/16 (our VPC CIDR)
- Description: Allow HTTP from within VPC
```

### 5.2 Database Security Group
```
Name: cloudlaunch-db-sg
Description: Security group for database servers
Inbound Rules:
- Type: MySQL/Aurora (3306)
- Source: 10.0.2.0/24 (the app subnet cidr)
- Description: Allow MySQL from app subnet only
```

**Security Design**: The database security group only allows access from the application subnet

## Step 6: Updating IAM User Permissions for VPC Access

I updated the IAM user policy to include VPC read permissions (shown in the policy above). This allows the `cloudlaunch-user` to view:
- VPC details and subnets
- Route tables and their associations
- Security groups and their rules
- Internet gateways and their attachments

---

# Architecture Summary

## Network Design
```
cloudlaunch-vpc (10.0.0.0/16)
├── cloudlaunch-public-subnet (10.0.1.0/24) - us-east-1a
│   └── → Internet Gateway (cloudlaunch-igw)
├── cloudlaunch-app-subnet (10.0.2.0/24) - us-east-1b
│   └── → Local traffic only
└── cloudlaunch-db-subnet (10.0.3.0/28) - us-east-1c
    └── → Local traffic only
```

## Security Implementation
- **Network Segmentation**: Three-tier architecture with proper isolation
- **Least Privilege Access**: IAM policies grant minimal required permissions
- **Defense in Depth**: Multiple security layers (IAM, Security Groups, Network ACLs)
- **HTTPS Enforcement**: CloudFront redirects all HTTP traffic to HTTPS
---

# Conclusion

This project successfully demonstrates a comprehensive AWS cloud infrastructure implementation, combining multiple core services into a secure, scalable solution. The CloudLaunch platform showcases:
- **Secure Static Website Hosting**: S3 and CloudFront integration with HTTPS enforcement and global content delivery
- **Access Control**: IAM policies implementing least privilege principles for secure resource access
- **Network Security Architecture**: Well-designed VPC with proper subnet isolation and security group configurations
- **Cost-Effective Deployment**: All resources deployed within AWS Free Tier while maintaining production-ready standards

This foundation provides a solid base for future expansion into compute resources, databases, and application deployment.