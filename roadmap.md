## **🎯 THE PROJECT: Global News Aggregator & Analytics Platform**

**Business Context:** A scalable platform that:
- Ingests news articles from multiple sources (RSS feeds, APIs)
- Processes and analyzes content (sentiment analysis, categorization)
- Stores structured data with fast query capabilities
- Serves content globally via secure APIs
- Sends personalized notifications to users
- Provides analytics dashboards for trends

**Why This Project?**
- Touches 80% of AWS services used in real production environments
- Forces you to make architectural decisions (when to use RDS vs DynamoDB?)
- Mimics actual interview case study questions
- Each phase is deployable and testable

---

## **📚 THE COMPLETE LEARNING PATH**

### **PHASE 1: THE FOUNDATION - VPC & IAM** *(Week 1-2)*
*"Security and networking are the bedrock. Get these wrong, and everything else is compromised."*

---

#### **1.1 AWS FUNDAMENTALS & CLOUD COMPUTING**

**Project Integration:** Understanding deployment models to choose the right service architecture (Serverless vs Container vs EC2).

**POC Task (Console):**
- Create an AWS Free Tier account
- Explore the AWS Console regions and availability zones
- Review the Shared Responsibility Model documentation
- Read the Well-Architected Framework whitepaper (just the 5 pillars overview)

**Production Task:**
Document your project architecture decisions using the Well-Architected Framework:
- Which pillar takes priority? (For news aggregator: Operational Excellence + Cost Optimization)
- Create a simple architecture diagram showing which services will be IaaS/PaaS/SaaS

**Expert Secrets:**
- **Hidden Cost:** Data transfer between AZs costs money. Same-region, same-AZ is free.
- **Multi-region strategy:** Most companies start single-region. Don't over-engineer early.
- **Service Limits:** Every service has soft/hard limits. Check them BEFORE architecture decisions.

**Interview Questions:**
1. *Conceptual:* "Explain the shared responsibility model. Who's responsible for patching OS on EC2 vs Lambda?"
2. *Scenario:* "Your company wants high availability but minimal costs. Would you deploy across 2 AZs or 3? Why?"
3. *Deep Dive:* "What happens to your application if an entire AZ goes down in your region?"

---

#### **1.2 VPC - YOUR PRIVATE CLOUD NETWORK**

**Project Integration:** The VPC hosts your EC2 instances (for initial testing), RDS databases, and provides network isolation for security.

**POC Task (Console):**
```
Goal: Create a basic VPC from scratch

1. Create VPC:
   - Name: news-aggregator-vpc
   - CIDR: 10.0.0.0/16 (65,536 IPs)
   
2. Create 2 Public Subnets:
   - Subnet-1: 10.0.1.0/24 (AZ: us-east-1a)
   - Subnet-2: 10.0.2.0/24 (AZ: us-east-1b)
   
3. Create Internet Gateway:
   - Attach to VPC
   
4. Create Route Table:
   - Add route: 0.0.0.0/0 → IGW
   - Associate with both public subnets
   
5. Test: Launch a t2.micro EC2 in Subnet-1 with auto-assign public IP
```

**Production Task (Console):**
```
Goal: Multi-tier network with public/private isolation

1. Create VPC: 10.0.0.0/16

2. Create 6 Subnets:
   Public Subnets (for Load Balancers):
   - 10.0.1.0/24 (us-east-1a)
   - 10.0.2.0/24 (us-east-1b)
   
   Private Subnets (for EC2/ECS):
   - 10.0.11.0/24 (us-east-1a)
   - 10.0.12.0/24 (us-east-1b)
   
   Database Subnets (for RDS):
   - 10.0.21.0/24 (us-east-1a)
   - 10.0.22.0/24 (us-east-1b)

3. Create NAT Gateways:
   - One in each public subnet (HA)
   - Allocate Elastic IPs

4. Create 3 Route Tables:
   - Public RT: 0.0.0.0/0 → IGW
   - Private RT (AZ-a): 0.0.0.0/0 → NAT-a
   - Private RT (AZ-b): 0.0.0.0/0 → NAT-b

5. Security Groups (we'll detail next):
   - ALB-SG (allow 80/443 from 0.0.0.0/0)
   - App-SG (allow traffic only from ALB-SG)
   - DB-SG (allow 3306 only from App-SG)

6. VPC Flow Logs:
   - Enable and send to CloudWatch Logs
```

**Expert Secrets:**
- **CIDR Math:** `/16` = 65K IPs, `/24` = 256 IPs. Always leave room for growth. Use `/20` for medium subnets (4096 IPs).
- **NAT Gateway Cost Trap:** $0.045/hour + $0.045/GB processed = ~$35/month PER NAT. For dev environments, use a single NAT in one AZ.
- **Reserved IPs:** AWS reserves 5 IPs per subnet (.0, .1, .2, .3, .255). A /28 subnet gives you 11 usable IPs, not 16.
- **VPC Peering Gotcha:** Non-transitive. If VPC-A peers with VPC-B, and VPC-B peers with VPC-C, VPC-A cannot reach VPC-C.

**Interview Questions:**
1. *Basic:* "What's the difference between a Security Group and a NACL?"
2. *Scenario:* "Your EC2 in a private subnet can't reach the internet. What are the 3 most common causes?"
3. *Advanced:* "You have a VPC with CIDR 10.0.0.0/16. How many /24 subnets can you create? What if you need /20 subnets?"
4. *Troubleshooting:* "An application in Subnet-A can't connect to RDS in Subnet-B, same VPC. Walk me through your debugging process."
5. *Cost:* "Your NAT Gateway bill is $500/month. What strategies would you use to reduce it?"

---

#### **1.3 IAM - IDENTITY & ACCESS MANAGEMENT**

**Project Integration:** IAM roles for Lambda functions, EC2 instance profiles, cross-service permissions, and securing your API.

**POC Task (Console):**
```
Goal: Understand users, groups, and policies

1. Create IAM User:
   - Name: news-app-developer
   - Enable console access
   - Attach policy: ReadOnlyAccess

2. Create IAM Group:
   - Name: Developers
   - Attach policies: AmazonS3FullAccess, AmazonEC2ReadOnlyAccess
   - Add user to group

3. Create Custom Policy:
   - Allow: s3:PutObject, s3:GetObject
   - Resource: arn:aws:s3:::news-aggregator-bucket/*
   - Deny: s3:DeleteBucket

4. Test: Login as the user and try operations
```

**Production Task (Console):**
```
Goal: Least-privilege roles for services

1. Lambda Execution Role:
   - Create role: news-ingestion-lambda-role
   - Trusted entity: lambda.amazonaws.com
   - Attach policies:
     * AWSLambdaBasicExecutionRole (CloudWatch Logs)
     * Custom inline policy:
       {
         "Version": "2012-10-17",
         "Statement": [
           {
             "Effect": "Allow",
             "Action": [
               "s3:GetObject",
               "s3:PutObject"
             ],
             "Resource": "arn:aws:s3:::news-raw-content/*"
           },
           {
             "Effect": "Allow",
             "Action": [
               "dynamodb:PutItem",
               "dynamodb:GetItem"
             ],
             "Resource": "arn:aws:dynamodb:us-east-1:*:table/NewsArticles"
           }
         ]
       }

2. EC2 Instance Role:
   - Create role: news-api-ec2-role
   - Trusted entity: ec2.amazonaws.com
   - Attach: AmazonSSMManagedInstanceCore (for Session Manager)
   - Custom policy: Read from specific S3 paths

3. Cross-Account Access:
   - Create role: external-analytics-role
   - Trusted entity: Another AWS account ID
   - Attach: Custom policy for specific DynamoDB read access

4. Enable MFA for root account and delete root access keys

5. Use AWS Organizations + SCPs (Service Control Policies):
   - Create OU: Production
   - SCP: Deny deletion of CloudTrail logs
```

**Expert Secrets:**
- **Policy Evaluation Logic:** Explicit Deny > Explicit Allow > Implicit Deny. An explicit deny ALWAYS wins.
- **Role Assumption:** When a Lambda assumes a role, it gets temporary credentials (15 min to 12 hours). These credentials are cached.
- **Resource-Based vs Identity-Based:** S3 bucket policies (resource-based) can grant cross-account access WITHOUT assuming roles. IAM policies (identity-based) cannot.
- **Permission Boundaries:** Advanced concept - they set the MAXIMUM permissions an entity can have, even if policies grant more.
- **Hidden Gotcha:** IAM is eventually consistent (except in us-east-1 for some operations). Wait 10-15 seconds after creating a role before using it.

**Interview Questions:**
1. *Basic:* "What's the difference between an IAM role and an IAM user?"
2. *Scenario:* "A Lambda function needs to read from S3 and write to DynamoDB. How do you grant it permissions?"
3. *Advanced:* "Explain how cross-account access works. What are the two ways to implement it?"
4. *Troubleshooting:* "Your EC2 instance can't access S3 despite having an IAM role with S3FullAccess. What could be wrong?"
5. *Security:* "How would you audit who deleted a critical S3 object 3 months ago?"
6. *Deep:* "What happens when both an S3 bucket policy and IAM policy grant access, but an SCP denies it?"

---

### **PHASE 2: COMPUTE & SCALING - THE APPLICATION TIER** *(Week 3-4)*

Now let's explore where your question-driven learning comes in...

**Before I detail Phase 2 (EC2, Auto Scaling, Load Balancers), let me ask:**

- Do you already know the basics of launching an EC2 instance?
- Have you worked with load balancers before (even outside AWS)?
- What confuses you most about Auto Scaling - the concept itself, or the configuration?

# PHASE 2: COMPUTE & SCALING - THE APPLICATION TIER *(Week 3-4)*

*"Compute is where your code runs. Mastering EC2 and Auto Scaling separates junior engineers from senior ones."*

---

## **2.1 EC2 - ELASTIC COMPUTE CLOUD**

**Project Integration:** EC2 will host your initial news aggregator API and background workers that fetch RSS feeds. Later, you'll transition to serverless, but EC2 teaches you the fundamentals of compute, networking, and troubleshooting.

### **POC Task (Console):**

```
Goal: Launch your first EC2 and understand instance fundamentals

1. Launch Instance Wizard:
   - Name: news-api-server-poc
   - AMI: Amazon Linux 2023
   - Instance type: t2.micro (free tier)
   - Key pair: Create new (news-app-key.pem) - DOWNLOAD IT
   - Network: Your VPC, Public Subnet
   - Auto-assign public IP: Enable
   - Security Group: Create new
     * Allow SSH (22) from My IP
     * Allow HTTP (80) from 0.0.0.0/0
   - Storage: 8GB gp3
   
2. User Data Script (paste in Advanced Details):
   #!/bin/bash
   yum update -y
   yum install -y httpd
   systemctl start httpd
   systemctl enable httpd
   echo "<h1>News Aggregator POC - $(hostname -f)</h1>" > /var/www/html/index.html

3. Connect:
   - SSH: ssh -i news-app-key.pem ec2-user@<PUBLIC-IP>
   - Test web server: curl http://<PUBLIC-IP>

4. Explore:
   - Instance details: Check private IP, availability zone
   - Monitoring: View CloudWatch metrics (CPU, Network)
   - Stop instance: See what happens to public IP
   - Start again: Public IP changed? Why?
```

**Challenge Question:** After stopping and starting your instance, the public IP changed. Your application connects to this IP. How would you solve this problem in production? (Think about it before moving to the production task.)

---

### **Production Task (Console):**

```
Goal: Production-grade EC2 with proper networking, storage, and security

1. Create Launch Template (not just an instance):
   - Name: news-api-template-v1
   - AMI: Amazon Linux 2023
   - Instance type: t3.medium
   - Key pair: news-app-key (or better: use Systems Manager Session Manager)
   - Network: 
     * VPC: news-aggregator-vpc
     * Subnet: DON'T SPECIFY (let ASG choose)
     * Security groups: app-tier-sg (from Phase 1)
   
   - IAM instance profile: news-api-ec2-role
   
   - Storage:
     * Root volume: 20GB gp3, encrypted (use default KMS key)
     * Add volume: 50GB gp3 for application logs, encrypted
     * Enable delete on termination
   
   - Advanced:
     * Detailed monitoring: Enable (1-min intervals)
     * Termination protection: Enable
     * User Data:
       #!/bin/bash
       yum update -y
       yum install -y docker
       systemctl start docker
       systemctl enable docker
       usermod -a -G docker ec2-user
       
       # Install CloudWatch agent
       wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
       rpm -U ./amazon-cloudwatch-agent.rpm
       
       # Pull and run your API container (we'll build this in ECS phase)
       docker pull nginx:latest
       docker run -d -p 80:80 --name news-api nginx
       
       # Send startup signal to CloudWatch
       aws cloudwatch put-metric-data --namespace NewsAggregator \
         --metric-name InstanceStartup --value 1 --region us-east-1

2. Elastic IP (for single instance testing):
   - Allocate Elastic IP
   - Associate with instance
   - Update DNS record to point to EIP

3. Enhanced Monitoring Setup:
   - Install CloudWatch agent (already in user data)
   - Configure custom metrics: memory usage, disk usage, application logs
   - Create CloudWatch dashboard for this instance

4. Backup Strategy:
   - Create AMI: "news-api-golden-image-v1"
   - Use AWS Backup: Daily snapshots, 7-day retention

5. Security Hardening:
   - Remove SSH key access: Use Systems Manager Session Manager instead
   - Update security group: Remove port 22, instance connects via SSM
   - Enable IMDSv2 (Instance Metadata Service v2):
     aws ec2 modify-instance-metadata-options \
       --instance-id i-xxx \
       --http-tokens required \
       --http-put-response-hop-limit 1
```

**Now pause here.** Before I give you the expert secrets:

**Think about this scenario:** You've deployed the instance above. Suddenly, your application is getting 10x traffic. What are THREE different ways you could scale this? (Don't just say "add more instances" - think about the mechanisms.)

---

### **Expert Secrets:**

**Instance Types Decoded:**
- **T3/T3a:** Burstable. Perfect for variable workloads. You earn CPU credits when idle, spend them when busy. **Gotcha:** Run out of credits = throttled to baseline (20% of vCPU for t3.medium).
- **M5/M6i:** General purpose. Predictable workloads. M6i has 15% better price/performance than M5.
- **C5/C6i:** Compute-optimized. CPU-intensive (video encoding, batch processing).
- **R5/R6i:** Memory-optimized. Databases, caches, real-time analytics.
- **G4/P3:** GPU instances. ML training/inference.

**Purchase Options Strategy:**
1. **On-Demand:** Default. Pay by the second. Use for unpredictable workloads.
2. **Reserved Instances (RI):** 1 or 3-year commitment = up to 72% discount. Buy for baseline capacity.
3. **Savings Plans:** More flexible than RI. Commit to $/hour spend (e.g., $10/hour for 1 year).
4. **Spot Instances:** Bid on unused capacity = up to 90% discount. Can be terminated with 2-min warning. Perfect for stateless workers.

**Hidden Knowledge:**
- **Placement Groups:** 
  - *Cluster:* Low-latency, same AZ (HPC applications)
  - *Spread:* Max 7 instances per AZ, different hardware (critical apps)
  - *Partition:* Groups of instances on separate hardware (Hadoop, Cassandra)

- **Nitro System:** Modern EC2 runs on Nitro hypervisor (better performance, more instance types, faster EBS). Some features ONLY work on Nitro instances (e.g., 64,000 EBS IOPS).

- **EBS vs Instance Store:**
  - EBS: Network-attached, persistent, survives stop/start, backup-able
  - Instance Store: Physically attached, ephemeral, loses data on stop, faster I/O
  - Rule: Use EBS for databases, instance store for temporary caches

- **EBS Volume Types:**
  - **gp3:** New default. $0.08/GB/month. 3000 IOPS baseline (can provision up to 16,000).
  - **gp2:** Old default. IOPS scale with size (3 IOPS per GB, max 16,000).
  - **io2 Block Express:** Provisioned IOPS. Up to 64,000 IOPS, 4000 MB/s throughput. $0.125/GB + $0.065/IOPS. For databases.
  - **st1:** Throughput-optimized HDD. Big data, data warehouses. Cannot be boot volume.

**The Metadata Service Secret:**
- Every EC2 can query `http://169.254.169.254/latest/meta-data/` for info about itself
- IMDSv1 (old): Simple HTTP GET = security risk (SSRF attacks)
- IMDSv2 (new): Requires PUT request with token = secure
- Best practice: Enforce IMDSv2 to prevent credential theft

**Cost Optimization Trick:**
```bash
# Check if your instance type is oversized
# SSH into instance and run:
top  # Check CPU usage over time
free -h  # Check memory usage

# Most instances run at <20% CPU. Downsize to save 50%.
```

---

### **Interview Questions:**

**Basic:**
1. "What's the difference between stopping and terminating an EC2 instance?"
2. "Explain the difference between Security Groups and NACLs. If both are configured, which one is checked first?"

**Scenario-Based:**
3. "Your application requires 100% uptime. How would you architect EC2 instances to achieve this?"
4. "You have a batch processing job that takes 4 hours to run, once per day. It's CPU-intensive but can be interrupted. What's the most cost-effective instance purchasing option?"
5. "Your web application runs on t3.medium instances. During a traffic spike, users report slow response times. You check CloudWatch and see CPU at 5%, network is fine. What's likely the issue?"

**Advanced:**
6. "Explain how Spot Instance interruptions work. How would you design a system to handle them gracefully?"
7. "Your database EC2 instance needs 30,000 IOPS consistently. Walk me through your EBS configuration and cost calculation."
8. "What's the difference between an AMI, a snapshot, and an instance template? When would you use each?"

**Troubleshooting:**
9. "You launched an EC2 with a user data script that installs software. The instance shows 'running' but the software isn't installed. How do you debug this?"
10. "An instance in a private subnet can't reach the internet through the NAT Gateway. The route table looks correct. What are the next 3 things you check?"

**Cost & Performance:**
11. "Your company spends $50,000/month on On-Demand instances that run 24/7. What's your optimization strategy?"
12. "How do you decide between scaling vertically (bigger instance) vs horizontally (more instances)?"

---

**Before we move to Auto Scaling, try this hands-on challenge:**

**Challenge Task:** 
Launch an EC2 instance that:
1. Runs in a private subnet (no public IP)
2. You can still access it via Systems Manager Session Manager
3. It can pull Docker images from Docker Hub
4. It sends custom metrics to CloudWatch (create a metric called "ApplicationHealth")

This combines VPC, IAM, and EC2 concepts. Try it in the console, and if you get stuck, walk me through where you're hitting issues. This is EXACTLY the kind of practical scenario interviewers love.

**Question for you:** Have you encountered Systems Manager Session Manager before? It's a game-changer for secure access. If not, I'll break it down in the Auto Scaling section.

---

## **2.2 AUTO SCALING - ELASTIC FLEET MANAGEMENT**

**Project Integration:** Your news aggregator will have variable traffic (morning spikes when people check news, lunch spikes, evening spikes). Auto Scaling ensures you have enough capacity without over-provisioning.

**Conceptual Foundation First:**

Before we dive into hands-on, let me ask you three questions to gauge your mental model:

1. **What do you think is the difference between "scaling out" vs "scaling up"?**

2. **If I told you to design a system that automatically adds servers when traffic increases, what metrics would you monitor to make that decision?** (Think beyond just CPU)

3. **Here's a tricky one:** An Auto Scaling Group launches 5 instances. All 5 pass health checks. Suddenly, one instance crashes. What should happen next, and how long should it take?

Take a moment to think through these. Your answers will help me calibrate the depth of explanation you need.

---

### **POC Task (Console):**

```
Goal: Create a basic Auto Scaling Group with manual scaling

1. Create Launch Template (if you didn't in EC2 section):
   - Name: news-api-asg-template
   - AMI: Your custom AMI or Amazon Linux 2023
   - Instance type: t3.micro
   - Key pair: news-app-key
   - Security group: app-tier-sg
   - User data: (simple web server from earlier)

2. Create Auto Scaling Group:
   - Name: news-api-asg-poc
   - Launch template: news-api-asg-template
   - VPC: news-aggregator-vpc
   - Subnets: Select BOTH private subnets (multi-AZ)
   - Load balancing: None (for now)
   - Health checks: EC2 (default)
   - Group size:
     * Desired: 2
     * Minimum: 1
     * Maximum: 4
   - Scaling policies: None (manual scaling)

3. Test Manual Scaling:
   - Observe: 2 instances launch automatically
   - Terminate one instance manually
   - Observe: ASG detects unhealthy instance, launches replacement
   - Change desired capacity to 3
   - Observe: New instance launches

4. Experiment:
   - What happens if you manually stop an instance?
   - What happens if you change the max to 1 (below current desired)?
```

**Reflection Questions:**
- The ASG launched instances in multiple subnets. Why is this important?
- When you terminated an instance, how long did it take for the replacement to launch?
- What would happen if one of your subnets became unavailable?

---

### **Production Task (Console):**

```
Goal: Production-ready ASG with dynamic scaling, health checks, and lifecycle hooks

1. Create Target Tracking Scaling Policy:
   
   First, let's create the ALB (we'll detail this in 2.3, but need it for ASG integration):
   
   a. Create Application Load Balancer:
      - Name: news-api-alb
      - Scheme: Internet-facing
      - Subnets: Both PUBLIC subnets
      - Security group: alb-sg (allow 80/443 from 0.0.0.0/0)
   
   b. Create Target Group:
      - Name: news-api-targets
      - Target type: Instances
      - Protocol: HTTP, Port: 80
      - VPC: news-aggregator-vpc
      - Health check:
        * Path: /health
        * Interval: 30 seconds
        * Healthy threshold: 2
        * Unhealthy threshold: 3
        * Timeout: 5 seconds

2. Create Production Auto Scaling Group:
   
   - Name: news-api-asg-prod
   - Launch template: news-api-template-v1
   - VPC: news-aggregator-vpc
   - Subnets: Both PRIVATE subnets (app tier)
   
   - Load balancing:
     * Attach to existing load balancer
     * Choose target group: news-api-targets
   
   - Health checks:
     * Type: ELB (not just EC2)
     * Grace period: 300 seconds (5 min for app to start)
   
   - Group size:
     * Desired: 3
     * Minimum: 2
     * Maximum: 10
   
   - Automatic scaling:
     * Target tracking policy:
       - Metric: Average CPU utilization
       - Target value: 50%
       - Instance warmup: 300 seconds
     
     * (Add second policy) Step scaling for burst traffic:
       - CloudWatch alarm: ALBRequestCount > 10,000
       - Action: Add 3 instances
       - Cooldown: 60 seconds

3. Advanced Configuration:
   
   a. Instance Refresh (for zero-downtime deployments):
      - When you update the launch template, trigger instance refresh
      - Min healthy percentage: 90%
      - Instance warmup: 300 seconds
   
   b. Lifecycle Hooks (for graceful shutdown):
      - Hook name: drain-connections-before-terminate
      - Lifecycle transition: Instance terminating
      - Heartbeat timeout: 300 seconds
      - Default result: CONTINUE
      
      Purpose: When scaling in, wait 5 minutes for connections to drain
   
   c. Termination Policies:
      - Order: OldestInstance (default)
      - Or use: OldestLaunchTemplate (replace old versions first)
   
   d. Suspended Processes:
      - During deployments, suspend: ReplaceUnhealthy
      - This prevents ASG from killing instances during brief restarts

4. Monitoring Setup:
   
   - Enable ASG metrics in CloudWatch:
     * GroupDesiredCapacity
     * GroupInServiceInstances
     * GroupTerminatingInstances
   
   - Create CloudWatch alarms:
     * Alarm: If InServiceInstances < DesiredCapacity for 5 minutes
     * Action: Send SNS notification to ops team
   
   - Create dashboard showing:
     * ASG capacity metrics
     * ALB request count
     * Target group healthy host count
     * Average CPU across all instances

5. Cost Optimization with Mixed Instance Policy:
   
   - Update ASG to use Spot Instances:
     * On-Demand base: 2 instances (always available)
     * On-Demand above base: 25% (for predictability)
     * Spot instances: 75% (for cost savings)
     * Instance types: t3.medium, t3a.medium, t3.large (diversified)
     * Spot allocation strategy: price-capacity-optimized
```

---

### **Expert Secrets:**

**Scaling Policy Deep Dive:**

1. **Target Tracking** (Recommended for most use cases):
   - You set a target (e.g., 50% CPU)
   - ASG automatically creates CloudWatch alarms
   - Scales out aggressively, scales in conservatively
   - Use for: CPU, Network, ALB request count, custom metrics

2. **Step Scaling** (For complex scenarios):
   - You define alarms and step adjustments
   - Example: 
     * 50-60% CPU: +1 instance
     * 60-80% CPU: +3 instances
     * 80%+ CPU: +5 instances
   - Use for: Non-linear scaling needs

3. **Scheduled Scaling** (For predictable patterns):
   - Schedule: Every weekday at 8 AM, set min=10
   - Schedule: Every weekday at 8 PM, set min=2
   - Use for: Known traffic patterns (business hours)

4. **Predictive Scaling** (ML-based):
   - ASG analyzes past traffic patterns
   - Scales proactively BEFORE traffic hits
   - Use for: Mature applications with predictable patterns

**The Hidden Complexity - Scaling Metrics:**

Most engineers only use CPU. Here are better options:

```
Bad: Average CPU utilization
Why: If 1 instance at 90% and 9 at 10%, average is 18% = no scaling

Better: Custom CloudWatch metric
Example: ApplicationBacklog (number of queued jobs)
         If backlog > 100, scale out
         If backlog < 20, scale in

Best: Multiple metrics with target tracking
- Primary: ALB target response time (scale if >200ms)
- Secondary: ALB request count per target (scale if >1000 req/target)
- Fallback: CPU (scale if >70%)
```

**Cooldown vs Warmup:**
- **Cooldown:** Time to WAIT before next scaling action (prevents flapping)
- **Warmup:** Time for new instance to BECOME productive (excludes from metrics)

Example: You set warmup=300s. A new instance launches at 1:00 PM.
- 1:00-1:05: Instance is warming up, not counted in ASG metrics
- 1:05: Instance now counts toward CPU average and request count

**Lifecycle Hooks - The Professional's Tool:**

```
Real-world scenario:
1. ASG decides to terminate instance i-12345
2. Lifecycle hook pauses termination
3. Lambda function triggered (via EventBridge)
4. Lambda:
   - Deregisters instance from service discovery
   - Waits for active connections to drain
   - Backs up logs to S3
   - Sends "CONTINUE" to complete termination
5. Instance terminates
```

Without this, connections get dropped = bad user experience.

**Spot Instance Strategies:**

```
Configuration:
- On-Demand baseline: 2 instances (always available)
- Spot instances: 8 instances (diversified across 3 instance types)
- Spot max price: Leave blank (pay current spot price, up to On-Demand price)

Spot Interruption Handling:
1. EC2 sends 2-minute warning
2. CloudWatch Events rule catches it
3. Lambda gracefully drains connections
4. ASG replaces with another Spot or On-Demand
```

**Cost Savings:** Spot can save 70-90%. For stateless apps (like your API), this is a no-brainer.

**The Math Behind ASG Decisions:**

```
Scenario: Target Tracking with 50% CPU

Current state:
- 4 instances
- Average CPU: 72%

Calculation:
Desired capacity = Current capacity × (Current metric / Target metric)
Desired capacity = 4 × (72 / 50)
Desired capacity = 5.76
Rounded: 6 instances

ASG scales out by 2 instances.
```

**Common Pitfalls:**

1. **Scaling In Too Aggressively:**
   - Default: ASG scales out fast, scales in slowly
   - Don't override this unless you understand the risk
   - Flapping (scale out, scale in, scale out) wastes money and causes instability

2. **Wrong Health Check Type:**
   - EC2 health check: Instance is running (OS level)
   - ELB health check: Application is responding to HTTP checks
   - ALWAYS use ELB health checks if you have an ALB

3. **Insufficient Capacity for Updates:**
   - You set min=2, max=2
   - You trigger instance refresh
   - ASG can't launch new instances (at max capacity)
   - Refresh fails
   - Solution: Temporarily increase max during deployments

---

### **Interview Questions:**

**Basic:**
1. "What's the difference between desired capacity, minimum capacity, and maximum capacity in an ASG?"
2. "Explain the difference between scaling out and scaling up."

**Scenario-Based:**
3. "Your Auto Scaling Group has 5 instances. All are healthy. You update the launch template with a new AMI. What happens to the existing instances?"
4. "Your application takes 3 minutes to start and begin serving traffic. How do you configure ASG to avoid premature health check failures?"
5. "During a traffic spike, your ASG keeps launching instances but they keep getting terminated. What could be wrong?"

**Advanced:**
6. "Explain the difference between target tracking, step scaling, and scheduled scaling. When would you use each?"
7. "Your application processes jobs from an SQS queue. How would you configure ASG to scale based on queue depth?"
8. "What's a lifecycle hook and when would you use one? Describe a real-world scenario."

**Troubleshooting:**
9. "Your ASG is configured with min=2, desired=3, max=5. You see only 2 instances running. What are the possible causes?"
10. "Instances are launching in your ASG but immediately getting terminated. Walk me through your debugging process."

**Cost & Architecture:**
11. "Your ASG runs 10 instances 24/7. How would you optimize costs without changing application code?"
12. "Explain how you'd design an ASG configuration that uses 80% Spot instances safely."

**Deep Dive:**
13. "If you have multiple scaling policies attached to one ASG, how does ASG decide which one to execute?"
14. "Your target tracking policy is set to 50% CPU, but instances are consistently running at 65%. Why isn't ASG scaling out more?"

---

## **2.3 ELASTIC LOAD BALANCER - TRAFFIC DISTRIBUTION**

I've already introduced ALB in the ASG section, but let's dive deeper.

**Project Integration:** The ALB sits in front of your Auto Scaling Group, distributing traffic across instances, performing health checks, and enabling zero-downtime deployments.

**Quick Knowledge Check:**

Before the hands-on, answer these:

1. **What's the difference between a Classic Load Balancer, Application Load Balancer, and Network Load Balancer?**

2. **If a target (EC2 instance) fails a health check, what happens to existing connections to that instance?**

3. **Can a Load Balancer route traffic to targets in different VPCs?**

Think through these, then I'll give you the detailed tasks...

---

### **POC Task (Console):**

```
Goal: Basic ALB routing traffic to 2 EC2 instances

1. Launch 2 EC2 Instances:
   - Use the same launch template
   - Both in public subnets (for POC simplicity)
   - User data that creates unique web pages:
     
     #!/bin/bash
     yum install -y httpd
     systemctl start httpd
     INSTANCE_ID=$(ec2-metadata --instance-id | cut -d " " -f 2)
     echo "<h1>Instance: $INSTANCE_ID</h1>" > /var/www/html/index.html

2. Create Application Load Balancer:
   - Name: news-api-alb-poc
   - Scheme: Internet-facing
   - IP address type: IPv4
   - Subnets: Select 2 public subnets (minimum required)
   - Security group: Create new
     * Inbound: HTTP (80) from 0.0.0.0/0

3. Create Target Group:
   - Name: news-api-tg-poc
   - Target type: Instances
   - Protocol: HTTP, Port: 80
   - VPC: Your VPC
   - Health check:
     * Protocol: HTTP
     * Path: /
     * Healthy threshold: 2 consecutive successes
   - Register targets: Select your 2 EC2 instances

4. Create Listener:
   - Protocol: HTTP
   - Port: 80
   - Default action: Forward to news-api-tg-poc

5. Test:
   - Get ALB DNS name: news-api-alb-poc-123456.us-east-1.elb.amazonaws.com
   - Refresh multiple times in browser
   - Observe: Traffic alternates between instances (round-robin)
   
6. Test Health Checks:
   - SSH into one instance: sudo systemctl stop httpd
   - Wait 60 seconds
   - Observe: All traffic goes to healthy instance
   - Restart: sudo systemctl start httpd
   - Wait 60 seconds
   - Observe: Traffic resumes to both instances
```

**Reflection:** Notice that you never connected to instance IPs directly. The ALB abstracted that away. This is why auto-scaling works seamlessly.

---

### **Production Task (Console):**

```
Goal: Enterprise-grade ALB with SSL, path routing, and advanced features

1. Create Production ALB:
   - Name: news-api-alb-prod
   - Scheme: Internet-facing
   - IP address type: IPv4
   - Subnets: 2 PUBLIC subnets (ALB must be in public subnets)
   - Security group: alb-sg
     * Inbound: 
       - HTTP (80) from 0.0.0.0/0
       - HTTPS (443) from 0.0.0.0/0
   
   - Attributes:
     * Deletion protection: Enable
     * Idle timeout: 60 seconds (default is 60)
     * Drop invalid headers: Enable
     * Access logs: Enable → S3 bucket: news-alb-logs-bucket

2. Request SSL Certificate (ACM):
   - Go to AWS Certificate Manager
   - Request public certificate
   - Domain: newsaggregator.yourname.com
   - Validation: DNS validation
   - Add CNAME records to Route 53 (we'll cover this in Phase 6)
   - Wait for certificate to be issued

3. Create Multiple Target Groups:
   
   a. API Target Group:
      - Name: news-api-tg
      - Protocol: HTTP, Port: 80
      - Health check: /api/health
      - Stickiness: Enable (1 hour duration)
   
   b. Admin Target Group:
      - Name: news-admin-tg
      - Protocol: HTTP, Port: 8080
      - Health check: /admin/health
      - Stickiness: Disabled
   
   c. WebSocket Target Group:
      - Name: news-ws-tg
      - Protocol: HTTP, Port: 80
      - Health check: /ws/health
      - Stickiness: Enable (required for WebSocket)
      - Attributes: Deregistration delay: 300 seconds

4. Configure Listeners with Advanced Routing:
   
   Listener 1: HTTP (Port 80)
   - Default action: Redirect to HTTPS (Port 443, Status 301)
   
   Listener 2: HTTPS (Port 443)
   - SSL certificate: Select from ACM
   - Security policy: ELBSecurityPolicy-TLS13-1-2-2021-06
   
   - Rules (evaluate in order):
     
     Rule 1: API traffic
     - IF: Path is /api/*
     - THEN: Forward to news-api-tg
     - Priority: 1
     
     Rule 2: Admin traffic (IP whitelist)
     - IF: Path is /admin/* AND Source IP is in 203.0.113.0/24
     - THEN: Forward to news-admin-tg
     - ELSE: Return fixed response (403 Forbidden)
     - Priority: 2
     
     Rule 3: WebSocket traffic
     - IF: Path is /ws/* AND HTTP header "Upgrade" is "websocket"
     - THEN: Forward to news-ws-tg
     - Priority: 3
     
     Rule 4: Health check
     - IF: Path is /health
     - THEN: Return fixed response (200 OK, "healthy")
     - Priority: 4
     
     Default Rule:
     - Forward to news-api-tg

5. Configure Health Checks (Fine-tuning):
   
   For news-api-tg:
   - Protocol: HTTP
   - Path: /api/health
   - Port: Traffic port
   - Healthy threshold: 2
   - Unhealthy threshold: 3
   - Timeout: 5 seconds
   - Interval: 30 seconds
   - Success codes: 200
   
   The math: Instance must pass 2 checks (2 × 30s = 1 minute) to become healthy
              Instance must fail 3 checks (3 × 30s = 90s) to become unhealthy

6. Enable Access Logs:
   - Create S3 bucket: news-alb-logs-yourname
   - Bucket policy: Allow ALB to write logs (AWS provides template)
   - Enable on ALB
   - Logs format: 
     https[timestamp] request_processing_time target_processing_time response_processing_time
     
   - Use logs for: Debugging, security analysis (Athena queries)

7. Set Up CloudWatch Alarms:
   
   a. Unhealthy Targets:
      - Metric: UnHealthyHostCount
      - Condition: > 0 for 2 datapoints within 5 minutes
      - Action: SNS topic → Ops team
   
   b. High Response Time:
      - Metric: TargetResponseTime
      - Condition: > 500ms (average) for 5 minutes
      - Action: SNS topic
   
   c. 5XX Errors:
      - Metric: HTTPCode_Target_5XX_Count
      - Condition: > 50 in 5 minutes
      - Action: SNS topic + Lambda (auto-remediate)

8. WAF Integration (Web Application Firewall):
   - Create WAF Web ACL: news-api-waf
   - Associate with ALB
   - Rules:
     * Rate limiting: 2000 requests per 5 min per IP
     * AWS Managed Rules: Core rule set
     * Geo-blocking: Block traffic from high-risk countries
     * SQL injection protection
     * XSS protection
```

---

### **Expert Secrets:**

**ALB vs NLB vs CLB:**

| Feature | Application LB (ALB) | Network LB (NLB) | Classic LB (CLB) |
|---------|---------------------|------------------|------------------|
| OSI Layer | Layer 7 (HTTP/HTTPS) | Layer 4 (TCP/UDP) | Layer 4 & 7 (legacy) |
| Routing | Path, Host, Header, Query | IP/Port only | Basic |
| WebSocket | ✅ | ✅ | ⚠️ (limited) |
| SSL Termination | ✅ | ✅

| Static IP | ❌ (DNS only) | ✅ (Elastic IP) | ❌ |
| Latency | ~50ms | ~100μs (ultra-low) | ~100ms |
| Price | $0.0225/hour | $0.0225/hour | $0.025/hour |
| Target Types | Instance, IP, Lambda | Instance, IP, ALB | Instance only |
| Use Case | Web apps, APIs, microservices | Gaming, IoT, extreme performance | Legacy (deprecated) |

**Choose ALB when:** You need HTTP/HTTPS features (path routing, SSL termination, WAF integration)
**Choose NLB when:** You need extreme performance, static IPs, or non-HTTP protocols (database connections, gaming)

---

**Connection Draining vs Deregistration Delay:**

```
Scenario: You have an instance processing requests, and you want to remove it.

Without deregistration delay:
1. ALB deregisters instance
2. In-flight requests get dropped = 500 errors
3. Users angry

With deregistration delay (300 seconds):
1. ALB marks instance as "draining"
2. No NEW connections sent to instance
3. EXISTING connections allowed to complete (up to 300s)
4. After 300s or when all connections close, instance deregistered
5. Zero downtime
```

**Default: 300 seconds. Tune based on your app:**
- API with quick requests: 30-60 seconds
- Long-polling connections: 300-600 seconds
- File uploads: 900 seconds (15 min max)

---

**Sticky Sessions (Session Affinity):**

```
Problem: User logs in → Session stored in instance memory → Next request goes to different instance → User logged out

Solution 1: Application-controlled stickiness
- ALB uses a cookie named AWSALB
- Routes user to same instance based on cookie
- Duration: 1 second to 7 days
- Use when: Simple session management

Solution 2: Application cookie stickiness
- Your app sets a custom cookie (e.g., JSESSIONID)
- ALB reads this cookie for routing decisions
- Use when: Complex session management

Best Solution: Don't use sticky sessions
- Store sessions in ElastiCache (Redis) or DynamoDB
- Any instance can serve any request
- Better for auto-scaling
```

---

**Cross-Zone Load Balancing:**

```
Scenario: 
- AZ-A: 3 instances
- AZ-B: 1 instance

WITHOUT cross-zone load balancing:
- 50% traffic to AZ-A → split among 3 instances (16.6% each)
- 50% traffic to AZ-B → 1 instance gets all (50%)
- Imbalanced!

WITH cross-zone load balancing (ALB default):
- Traffic evenly distributed across ALL 4 instances (25% each)
- Balanced!

Cost: ALB has this ENABLED by default at no extra charge
      NLB charges $0.01/GB for cross-zone traffic
```

---

**Health Check Deep Dive:**

```
Advanced health check configuration:

Path: /api/health
Response: Must check dependencies

Good health endpoint:
{
  "status": "healthy",
  "checks": {
    "database": "connected",
    "cache": "connected",
    "disk_space": "ok"
  }
}

Bad health endpoint:
Simply returns 200 OK (doesn't check dependencies)

Advanced pattern: Separate shallow vs deep health checks
- /health (shallow): Returns 200 if app is running (for ALB)
- /health/deep (deep): Checks database, cache, etc. (for monitoring)
```

**Health check math matters:**

```
Configuration:
- Interval: 30 seconds
- Timeout: 5 seconds
- Healthy threshold: 2
- Unhealthy threshold: 3

Time to become healthy: 2 × 30s = 60 seconds minimum
Time to become unhealthy: 3 × 30s = 90 seconds minimum

Impact on auto-scaling:
- Set ASG health check grace period > 60 seconds
- Otherwise, ASG terminates instances before they become healthy
```

---

**SSL/TLS Best Practices:**

```
Security Policy: ELBSecurityPolicy-TLS13-1-2-2021-06
- TLS 1.3 (preferred)
- TLS 1.2 (fallback)
- Disables TLS 1.0/1.1 (insecure)

Certificate Management:
1. Use ACM (AWS Certificate Manager) - FREE
2. Auto-renewal (before 60-day expiry)
3. Can import third-party certs if needed

SNI (Server Name Indication):
- One ALB can have MULTIPLE SSL certificates
- Routes based on hostname:
  * api.newsaggregator.com → Cert A
  * admin.newsaggregator.com → Cert B
  * ws.newsaggregator.com → Cert C
```

---

**Access Logs - Hidden Goldmine:**

```
Log format (partial):
https 2025-10-04T10:23:45.678901Z app/news-api-alb/a1b2c3d4e5f6 
203.0.113.42:54321 10.0.1.15:80 0.001 0.003 0.000 200 200 
1024 512 "GET https://api.newsaggregator.com/articles HTTP/1.1"

Fields explained:
- 0.001: Request processing time (ALB overhead)
- 0.003: Target processing time (your app's response time)
- 0.000: Response processing time (ALB sending response)

Total latency: 0.001 + 0.003 + 0.000 = 4ms

Query with Athena:
CREATE EXTERNAL TABLE alb_logs (
  type string,
  time string,
  elb string,
  client_ip string,
  target_ip string,
  request_processing_time double,
  target_processing_time double,
  response_processing_time double,
  elb_status_code string,
  target_status_code string
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
WITH SERDEPROPERTIES (
  'serialization.format' = '1',
  'input.regex' = '([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*).*'
)
LOCATION 's3://news-alb-logs-yourname/';

-- Find slowest requests:
SELECT client_ip, target_processing_time, request
FROM alb_logs
WHERE target_processing_time > 1.0
ORDER BY target_processing_time DESC
LIMIT 100;

-- Find most active IPs (potential DDoS):
SELECT client_ip, COUNT(*) as request_count
FROM alb_logs
WHERE time > '2025-10-04'
GROUP BY client_ip
ORDER BY request_count DESC
LIMIT 50;
```

---

**Advanced Routing Scenarios:**

```
1. A/B Testing (Traffic Splitting):
   - Rule: Forward 90% to news-api-tg-v1
   - Rule: Forward 10% to news-api-tg-v2 (canary deployment)

2. Weighted Routing (Blue/Green):
   - During deployment:
     * 100% to blue (old version)
   - Start cutover:
     * 50% to blue, 50% to green (new version)
   - If green is stable:
     * 0% to blue, 100% to green
   - If green has issues:
     * Instant rollback to 100% blue

3. Host-based Routing:
   - IF host-header = api.newsaggregator.com → API target group
   - IF host-header = admin.newsaggregator.com → Admin target group

4. Header-based Routing (Feature Flags):
   - IF header "X-Beta-User" = "true" → Beta target group
   - ELSE → Production target group

5. Query String Routing:
   - IF query string contains "version=2" → V2 target group
   - ELSE → V1 target group
```

---

**Connection Limits & Quotas:**

```
ALB Limits (soft limits, can be increased):
- Targets per ALB: 1,000
- Targets per availability zone: 500
- Listeners per ALB: 50
- Rules per listener: 100
- Certificates per ALB: 25

Connection behavior:
- ALB maintains connection pools to targets
- New connections: ~50ms latency (SSL handshake)
- Kept-alive connections: ~5ms latency
- ALB can handle 1000s of connections per second

Tuning for high traffic:
- Enable HTTP/2 (multiplexes requests over single connection)
- Enable connection keep-alive on targets
- Increase idle timeout if needed (default 60s)
```

---

**WAF Integration Strategies:**

```
Rate Limiting Rule:
- Block IP if > 2000 requests in 5 minutes
- Prevents DDoS and brute force attacks

Cost consideration:
- WAF costs: $5/month + $1/rule/month + $0.60/million requests
- For 10M requests/month with 5 rules: $5 + $5 + $6 = $16/month

Advanced: Use AWS Shield Standard (FREE) + WAF
- Shield Standard: DDoS protection at network layer
- WAF: Application layer protection
- Combined: Comprehensive defense
```

---

**Cost Optimization Tricks:**

```
ALB Pricing (us-east-1):
- $0.0225 per hour (~$16/month)
- $0.008 per LCU-hour (Load Balancer Capacity Unit)

LCU = max of:
- 25 new connections/sec
- 3,000 active connections/min
- 1 GB/hour processed (for HTTP)
- 1,000 rule evaluations/sec

Example calculation:
App processes:
- 50 new connections/sec = 2 LCUs
- 5,000 active connections = 1.67 LCUs
- 2 GB/hour = 2 LCUs
- 500 rule evaluations/sec = 0.5 LCUs

You pay for: max(2, 1.67, 2, 0.5) = 2 LCUs/hour
Cost: 2 × $0.008 × 730 hours = $11.68/month
Total: $16 (base) + $11.68 (LCU) = $27.68/month

Optimization:
- Reduce rule complexity (fewer evaluations)
- Use connection pooling (fewer new connections)
- Compress responses (less data transferred)
```

---

**The Secret Weapon: Lambda as ALB Target:**

```
Use case: Serverless API without API Gateway

Setup:
1. Create Lambda function: news-api-lambda
2. Add ALB as trigger
3. Lambda receives ALB events in this format:

{
  "requestContext": {
    "elb": {
      "targetGroupArn": "arn:aws:..."
    }
  },
  "httpMethod": "GET",
  "path": "/api/articles",
  "queryStringParameters": {"category": "tech"},
  "headers": {
    "accept": "application/json",
    "x-forwarded-for": "203.0.113.42"
  },
  "body": null
}

4. Lambda must return:

{
  "statusCode": 200,
  "statusDescription": "200 OK",
  "headers": {
    "Content-Type": "application/json"
  },
  "body": "{\"articles\": [...]}"
}

Benefits over API Gateway:
- Lower latency (no extra hop)
- Lower cost (no API Gateway fees)
- Simpler architecture

Trade-off:
- Less features (no request validation, no API keys)
```

---

### **Interview Questions:**

**Basic:**
1. "What's the difference between Application Load Balancer and Network Load Balancer? When would you use each?"
2. "Explain how ALB health checks work. What happens when a target fails a health check?"

**Scenario-Based:**
3. "You have an ALB routing to 5 targets. One target is responding slowly (5 seconds per request). How does this affect the overall system?"
4. "Your application requires users to stay connected to the same instance for the duration of their session. How would you configure this?"
5. "You need to perform a zero-downtime deployment. Walk me through your ALB and ASG configuration."

**Advanced:**
6. "Explain the difference between connection draining and deregistration delay. Why does deregistration delay matter for long-running connections?"
7. "You want to route traffic based on URL path: /api/* to one target group, /admin/* to another, but only allow /admin/* from specific IPs. How do you configure this?"
8. "Your ALB is showing high latency in CloudWatch, but your application metrics show fast response times. Where is the bottleneck?"

**Troubleshooting:**
9. "Users are getting 502 Bad Gateway errors from your ALB. Walk me through your debugging process."
10. "Your ALB is in public subnets, targets in private subnets. Targets show as unhealthy. What could be wrong?"
11. "After deploying a new version, 50% of requests succeed and 50% fail with 500 errors. What's likely happening?"

**Security:**
12. "How do you prevent DDoS attacks against your ALB? List 3 strategies."
13. "Explain SSL termination. What are the security implications of terminating SSL at the ALB vs at the target?"

**Cost & Performance:**
14. "Your ALB bill is $500/month. How do you determine what's driving the cost and optimize it?"
15. "What's an LCU (Load Balancer Capacity Unit)? How is it calculated and billed?"

**Deep Dive:**
16. "Explain cross-zone load balancing. When would you disable it and why?"
17. "You have WebSocket connections that last 30 minutes. How do you configure ALB to handle this properly?"
18. "Can you have multiple SSL certificates on one ALB? How does the ALB decide which certificate to use?"

---

## **Real-World Challenge Task:**

**Challenge:** Set up a complete production-grade load-balanced environment:

1. **Create the full stack:**
   - VPC with public/private subnets
   - Auto Scaling Group (3 instances minimum) in private subnets
   - ALB in public subnets
   - Security groups allowing ONLY ALB → ASG communication

2. **Implement path-based routing:**
   - `/api/*` → API target group
   - `/health` → Return fixed 200 response
   - Everything else → Return 404

3. **Enable SSL:**
   - Request ACM certificate
   - Configure HTTPS listener
   - Redirect HTTP → HTTPS

4. **Monitoring:**
   - CloudWatch alarm for unhealthy targets
   - CloudWatch alarm for high 5xx errors
   - Enable access logs to S3

5. **Test scenarios:**
   - Stop httpd on one instance → Verify traffic routes around it
   - Trigger ASG scale-out → Verify new instance registers with ALB
   - Generate load → Verify auto-scaling works

**Bonus:** Configure a blue-green deployment where you can switch traffic between two target groups with different app versions.

---

**Before we move to Phase 3 (Storage & Databases), let me check in:**

1. **Hands-on Progress:** Have you completed the POC tasks for EC2, ASG, and ALB? Any blockers?

2. **Depth Preference:** Is this level of detail good, or would you like:
   - More hands-on tasks with less theory?
   - More interview questions?
   - More troubleshooting scenarios?

3. **Timeline Reality Check:** We've covered a LOT. Realistically:
   - Phase 1 (VPC/IAM): 1-2 weeks of hands-on
   - Phase 2 (Compute): 1-2 weeks
   - That's already a month of solid learning

   Should we:
   - **Option A:** Continue this detailed pace through all phases (12-16 weeks total)?
   - **Option B:** Speed up remaining phases with focused POC/Production tasks?
   - **Option C:** You tell me your deadline and we'll calibrate accordingly?

Also, quick question: **Are you more interested in becoming an AWS Solutions Architect or an AWS DevOps Engineer?** The answer changes how we prioritize Phase 5 (Lambda/Serverless) vs IaC/CI-CD integration.

Excellent! I can see you're serious about this journey. Given your DevOps background and the fact that you want to continue at this depth, let me proceed with **Phase 3: Storage & Databases**.

But first, let me do something important for your learning...

---

## **🎯 KNOWLEDGE CHECKPOINT - Phase 2 Review**

Before we dive into storage, I want you to actively engage with what you've learned. This isn't a test - it's about **solidifying your mental model** before we build on top of it.

**Take 5 minutes and answer these for yourself** (you don't need to write them out to me unless you want feedback, but thinking through them is crucial):

### **Mental Model Check:**

1. **The Big Picture:**
   - Close your eyes and visualize: A user request comes in from the internet. Draw the path it takes through VPC → ALB → EC2. What happens at each hop?
   - Where are the security checkpoints? (Hint: There are at least 4)

2. **The "Why" Questions:**
   - WHY do we put ALB in public subnets but EC2 in private subnets? What would break if we reversed this?
   - WHY does ASG use multiple subnets across different AZs? What specific failure scenario does this protect against?
   - WHY does deregistration delay exist? What happens to users if we set it to 0?

3. **The Trade-off Questions:**
   - If I gave you $1000/month for infrastructure, how would you split it between: instance size vs instance count vs storage vs NAT Gateway?
   - When would you choose 2 large instances vs 4 small instances? What factors matter?

**Be honest with yourself:** If any of these feel fuzzy, that's your signal to go back and do the hands-on POC tasks. Theory without practice creates illusion of knowledge.

---

Alright, let's continue. Here's the thing about **Phase 3** - this is where your **DevOps mindset becomes a superpower**. Storage and databases are where:
- Small decisions = massive cost differences
- Wrong choices = performance bottlenecks that no amount of scaling can fix
- Architecture mistakes = painful migrations later

---

# **PHASE 3: STORAGE & DATABASES** *(Week 5-7)*

*"Compute is temporary, storage is forever. Choose wisely."*

---

## **3.1 S3 - SIMPLE STORAGE SERVICE**

**Project Integration:** S3 will store:
- Raw news articles (JSON/XML from RSS feeds)
- Processed content (cleaned, analyzed)
- User-uploaded images/videos
- Application logs and backups
- Static assets served via CloudFront

**But before we jump into tasks, let me ask you something critical:**

### **🤔 Thought Exercise:**

You need to store 1 million news articles. Each article is ~50KB.

**Three approaches:**
1. **One S3 bucket, flat structure:** `articles/article-1.json`, `articles/article-2.json`, ...
2. **One S3 bucket, hierarchical:** `articles/2025/10/04/article-1.json`
3. **RDS database:** Store articles as rows in PostgreSQL

**Question for you:** What factors would influence your choice? Think about:
- Query patterns (how will you retrieve articles?)
- Cost (compute vs storage)
- Scalability (what if it becomes 100 million articles?)
- Performance (latency requirements)

Take a moment. What would YOU choose and why?

---

*(I'll wait for your thought process, but I'll also continue so you can see the full picture...)*

The answer is: **It depends on your access patterns.** This is the #1 principle of AWS architecture.

- **If you access articles by ID:** S3 with hierarchical structure
- **If you query articles by date, category, author:** Database (DynamoDB or RDS)
- **If you do full-text search:** OpenSearch + S3 (best of both worlds)

---

### **POC Task (Console):**

```
Goal: Understand S3 basics - buckets, objects, permissions

1. Create S3 Bucket:
   - Name: news-aggregator-raw-[yourname]-[random] 
     (must be globally unique)
   - Region: us-east-1
   - Block Public Access: Keep ENABLED (default)
   - Versioning: Disabled (for now)
   - Encryption: SSE-S3 (default encryption)

2. Upload Objects:
   - Create a test file: article-1.json
   {
     "id": "article-1",
     "title": "AWS S3 Deep Dive",
     "content": "S3 is object storage...",
     "published": "2025-10-04"
   }
   
   - Upload to: s3://news-aggregator-raw-yourname/articles/2025/10/04/article-1.json
   - Add metadata: Content-Type: application/json
   - Add tags: Environment=POC, Project=NewsAggregator

3. Explore Object Properties:
   - Click on uploaded object
   - Note: Object URL, ARN, ETag, Size
   - Try to access object URL in browser → Access Denied (good!)

4. Generate Presigned URL:
   - Select object → Actions → Share with presigned URL
   - Expiration: 5 minutes
   - Copy URL and open in browser → Success!
   - Wait 5 minutes → Access Denied (expired)

5. Create Folder Structure:
   articles/
     2025/
       10/
         04/
           article-1.json
           article-2.json
     raw/
     processed/

   Note: "Folders" in S3 are just prefixes in object keys

6. Test Permissions:
   - Create IAM user: s3-readonly-user
   - Attach policy: AmazonS3ReadOnlyAccess
   - Try to upload with this user → Denied
   - Try to download → Success
```

**Reflection Question:** You created "folders" in S3. But S3 is object storage, not file storage. What does this mean technically? How is `articles/2025/10/04/article-1.json` actually stored?

---

### **Production Task (Console):**

```
Goal: Production S3 with lifecycle policies, encryption, replication, and access control

1. Create Production Bucket Architecture:

   a. Raw Content Bucket:
      - Name: news-prod-raw-content-[random]
      - Region: us-east-1
      - Versioning: ENABLED
      - Encryption: SSE-KMS with custom CMK
      - Object Lock: Enabled (compliance mode for audit trail)
      - Tags: Environment=Production, DataClassification=Sensitive
   
   b. Processed Content Bucket:
      - Name: news-prod-processed-content-[random]
      - Region: us-east-1
      - Versioning: ENABLED
      - Encryption: SSE-S3
      - Tags: Environment=Production
   
   c. Public Assets Bucket (for CloudFront):
      - Name: news-prod-public-assets-[random]
      - Region: us-east-1
      - Versioning: DISABLED
      - Encryption: SSE-S3
      - Block Public Access: DISABLED (but use CloudFront OAI)

2. Create Custom KMS Key:
   - Go to KMS → Create key
   - Key type: Symmetric
   - Alias: news-content-encryption-key
   - Key administrators: Your IAM user
   - Key users: Lambda execution roles, EC2 instance roles
   - Note: KMS key ARN (you'll need it for IAM policies)

3. Configure Lifecycle Policies (news-prod-raw-content):
   
   Rule 1: Archive old articles
   - Name: archive-old-content
   - Scope: Prefix: articles/
   - Transitions:
     * After 30 days → S3 Intelligent-Tiering
     * After 90 days → S3 Glacier Instant Retrieval
     * After 365 days → S3 Glacier Deep Archive
   - Expiration: After 2555 days (7 years for compliance)
   
   Rule 2: Clean up incomplete multipart uploads
   - Name: cleanup-incomplete-uploads
   - Scope: All objects
   - Abort incomplete multipart uploads: After 7 days
   
   Rule 3: Delete old versions
   - Name: cleanup-old-versions
   - Scope: All objects
   - Noncurrent version transitions:
     * After 30 days → S3 Glacier
   - Permanently delete noncurrent versions: After 90 days

4. Set Up S3 Bucket Policies:
   
   Raw Content Bucket Policy:
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "AllowLambdaIngestion",
         "Effect": "Allow",
         "Principal": {
           "AWS": "arn:aws:iam::YOUR-ACCOUNT:role/news-ingestion-lambda-role"
         },
         "Action": [
           "s3:PutObject",
           "s3:PutObjectAcl"
         ],
         "Resource": "arn:aws:s3:::news-prod-raw-content-*/articles/*"
       },
       {
         "Sid": "DenyUnencryptedObjectUploads",
         "Effect": "Deny",
         "Principal": "*",
         "Action": "s3:PutObject",
         "Resource": "arn:aws:s3:::news-prod-raw-content-*/*",
         "Condition": {
           "StringNotEquals": {
             "s3:x-amz-server-side-encryption": "aws:kms"
           }
         }
       },
       {
         "Sid": "DenyInsecureTransport",
         "Effect": "Deny",
         "Principal": "*",
         "Action": "s3:*",
         "Resource": [
           "arn:aws:s3:::news-prod-raw-content-*",
           "arn:aws:s3:::news-prod-raw-content-*/*"
         ],
         "Condition": {
           "Bool": {
             "aws:SecureTransport": "false"
           }
         }
       }
     ]
   }

5. Enable S3 Event Notifications:
   
   On news-prod-raw-content bucket:
   - Event: s3:ObjectCreated:*
   - Prefix: articles/
   - Destination: SNS topic OR Lambda function
   - Use case: Trigger processing pipeline when new article uploaded

6. Configure Cross-Region Replication (CRR):
   
   - Create replica bucket: news-prod-raw-content-replica-[random]
   - Region: us-west-2 (different region for DR)
   - Enable versioning on replica bucket
   
   - Create replication rule on source bucket:
     * Name: replicate-all-content
     * Status: Enabled
     * Scope: All objects
     * Destination: replica bucket
     * IAM role: Create new role (S3 will create automatically)
     * Replication options:
       - Replicate objects encrypted with KMS: Yes
       - Replica storage class: Same as source
       - Replication time control: Enabled (15-min SLA)
   
   - Test: Upload object to source → Check replica after 15 min

7. Enable Access Logging:
   
   - Create logging bucket: news-prod-s3-access-logs-[random]
   - Enable server access logging on all production buckets
   - Target: news-prod-s3-access-logs-[random]
   - Prefix: raw-content-logs/ or processed-content-logs/
   
   - Logs format:
     [Owner] [Bucket] [Time] [Remote IP] [Requester] [Operation] [Key] [HTTP status] [Error Code]

8. Set Up S3 Storage Lens:
   
   - Create dashboard: news-aggregator-storage-lens
   - Scope: Account-level
   - Metrics: 
     * Storage metrics (total size, object count)
     * Cost metrics (storage class distribution)
     * Data protection metrics (encryption, versioning compliance)
     * Access patterns
   - Export to S3: news-prod-storage-lens-data-[random]

9. Configure S3 Inventory:
   
   - Schedule: Daily
   - Output: CSV
   - Destination: news-prod-inventory-[random]
   - Fields: Size, Last modified date, Storage class, Encryption status
   - Use case: Compliance audits, cost analysis

10. Implement S3 Object Lock (for compliance):
    
    On news-prod-raw-content bucket:
    - Object Lock mode: Compliance (cannot be deleted, even by root)
    - Default retention: 7 years
    - Legal hold: Enable for objects under investigation
    
    Use case: Regulatory compliance, forensics

11. Cost Optimization - S3 Intelligent-Tiering:
    
    - Enable on processed-content bucket
    - Archive configuration:
      * After 90 days of no access → Archive Access Tier
      * After 180 days → Deep Archive Access Tier
    - Monitoring fee: $0.0025 per 1000 objects (small cost for big savings)
```

**Critical Thinking Question:** You've set up lifecycle policies to move objects to Glacier after 90 days. But what if a user requests an article that's in Glacier? How long would retrieval take and how much would it cost? How would you architect your application to handle this?

---

### **Expert Secrets:**

**S3 is NOT a File System:**
```
Common mistake:
s3://bucket/folder1/folder2/file.txt

Reality:
- Object key: "folder1/folder2/file.txt"
- No folders exist
- S3 is a flat namespace with 4 billion objects per bucket

Why it matters:
- "List all files in folder1/folder2/" → Scans ALL objects, filters by prefix
- Listing 1 million objects with prefix → SLOW
- Better: Use manifest files or database index
```

**S3 Performance Secrets:**

```
Old wisdom (pre-2018): Randomize key prefixes for performance
- articles/a1b2/article-1.json
- articles/c3d4/article-2.json

New reality (post-2018): No longer needed!
- S3 now auto-scales to 3,500 PUT/5,500 GET requests per second PER PREFIX
- You can use natural prefixes: articles/2025/10/04/

Exception: If you need >3,500 PUT/s, shard across multiple prefixes:
- articles-shard-0/
- articles-shard-1/
- articles-shard-2/
```

**Storage Class Cost Breakdown (per GB/month):**

| Storage Class | Cost | Retrieval | Use Case |
|--------------|------|-----------|----------|
| S3 Standard | $0.023 | Free | Hot data, <30 days |
| S3 Intelligent-Tiering | $0.023 + $0.0025/1K objects | Free | Unknown access patterns |
| S3 Standard-IA | $0.0125 | $0.01/GB | Warm data, >30 days |
| S3 One Zone-IA | $0.01 | $0.01/GB | Non-critical, reproducible |
| S3 Glacier Instant | $0.004 | $0.03/GB | Archive, instant access |
| S3 Glacier Flexible | $0.0036 | $0.05/GB | Archive, 1-5 min retrieval |
| S3 Glacier Deep Archive | $0.00099 | $0.02/GB | Compliance, 12-hour retrieval |

**Cost Optimization Example:**
```
Scenario: 1 TB of news articles
- Access pattern: 10% accessed frequently, 90% rarely

Bad approach: All in S3 Standard
Cost: 1000 GB × $0.023 = $23/month

Good approach: Intelligent-Tiering
- Hot tier (100 GB): $2.30
- Archive tier (900 GB): $0.90
- Monitoring: 1M objects × $0.0025 = $2.50
Total: $5.70/month

Savings: 75%
```

**Versioning Hidden Costs:**

```
Problem: You enable versioning, upload 1 GB file, modify it 10 times

Storage cost:
- 1 GB (current version) = $0.023
- 10 GB (old versions) = $0.230
Total: $0.253/month

Solution: Lifecycle policy to delete old versions after 30 days
```

**Multipart Upload - The Professional's Secret:**

```
When to use:
- File >100 MB: Recommended
- File >5 GB: Required (single PUT has 5 GB limit)

Benefits:
- Parallel uploads (faster)
- Resume failed uploads
- Upload while creating (streaming)

How it works:
1. Initiate multipart upload
2. Upload parts (5 MB to 5 GB each, up to 10,000 parts)
3. Complete multipart upload (S3 assembles parts)

Gotcha: Incomplete multipart uploads COST MONEY
- Solution: Lifecycle policy to abort after 7 days
```

**Encryption Confusion Clarified:**

```
SSE-S3 (Server-Side Encryption with S3-managed keys):
- AWS manages keys
- Free
- Encryption at rest only
- Good for: Non-sensitive data

SSE-KMS (Server-Side Encryption with KMS):
- You control keys (can audit, rotate, delete)
- Cost: $1/key/month + $0.03/10K requests
- Encryption at rest + access control
- Good for: Sensitive data, compliance
- Gotcha: KMS has request limits (5,500/s decrypt)

SSE-C (Server-Side Encryption with Customer-provided keys):
- You provide key with every request
- Free
- You manage key rotation
- Good for: Maximum control, rarely used

Client-Side Encryption:
- You encrypt before upload
- AWS never sees unencrypted data
- Good for: Extreme sensitivity
```

**S3 Select - The Hidden Gem:**

```
Problem: You have 1 GB CSV file, need 1 row

Bad approach:
s3.get_object() → Download 1 GB → Parse → Find row
Cost: $0.09 (data transfer) + Time: 30 seconds

Good approach: S3 Select
SELECT * FROM s3object WHERE id = '12345'
Cost: $0.002 (scanned) + $0.0007 (returned) = $0.003
Time: 2 seconds

Savings: 97% cost, 93% time
```

**Bucket Naming Gotchas:**

```
Rules:
- 3-63 characters
- Lowercase only
- No underscores, only hyphens
- Cannot look like IP address (192.168.1.1)
- Must be globally unique

Pro tip: Include account ID or random string
- news-aggregator-123456789012-raw
- news-aggregator-a7f2-processed

Why: Avoids conflicts, easier to identify in logs
```

---

### **Interview Questions:**

**Basic:**
1. "What's the difference between S3 and EBS? When would you use each?"
2. "Explain S3's consistency model. What guarantees does S3 provide?"
3. "What's the maximum size of a single S3 object?"

**Scenario-Based:**
4. "You need to store 10 TB of log files that are accessed once per month for compliance. Which S3 storage class would you choose and why?"
5. "Your application uploads 1 million images per day (each 1 MB). Users access 10% of images within 24 hours, then rarely again. Design your S3 strategy."
6. "A user accidentally deleted 100 critical files from S3. How do you recover them?"

**Advanced:**
7. "Explain the difference between S3 bucket policies and IAM policies. When would you use each? Can they conflict?"
8. "You have a 10 GB file in S3. Describe three different ways to download it and the trade-offs of each approach."
9. "How does S3 replication work? What's the difference between CRR and SRR? What are the use cases for each?"

**Troubleshooting:**
10. "A Lambda function can read from an S3 bucket but cannot write. The IAM role has S3FullAccess. What could be wrong?"
11. "You're getting 503 SlowDown errors from S3. What's causing this and how do you fix it?"
12. "Objects are being uploaded to S3 but aren't encrypted, despite your bucket policy requiring encryption. Why might this happen?"

**Cost & Performance:**
13. "Your S3 bill is $5,000/month for 100 TB. Walk me through your optimization strategy."
14. "You need to serve 1 million GET requests per second from S3. Is this possible? How would you architect it?"
15. "Explain S3 request costs. How much does it cost to list 1 million objects? To download 1 TB?"

**Security:**
16. "How do you prevent accidental public exposure of S3 buckets? What are the layers of defense?"
17. "Explain presigned URLs. How are they generated and what are the security implications?"
18. "What's the difference between bucket ACLs and bucket policies? Which should you use in 2025?"

**Deep Dive:**
19. "Explain how S3 achieves 99.999999999% (11 nines) durability. What does this actually mean?"
20. "You need to grant cross-account access to specific S3 objects. What are three ways to implement this?"

---

**Before I continue to RDS, here's a hands-on challenge for you:**

### **🎯 Real-World S3 Challenge:**

**Build a complete S3-based content pipeline:**

1. **Create the architecture:**
   - Raw content bucket with event notifications
   - Lambda triggered on S3 upload
   - Lambda processes file and writes to processed bucket
   - Lifecycle policies on both buckets
   - CloudWatch logs for debugging

2. **Test scenarios:**
   - Upload a JSON file → Lambda processes it
   - Upload a 100 MB file → Verify multipart upload
   - Delete an object → Verify versioning keeps it
   - Wait 31 days (or change system clock on a test object) → Verify lifecycle transition

3. **Monitoring:**
   - CloudWatch metric: Number of objects in bucket
   - CloudWatch alarm: If bucket size > 1 TB
   - S3 Storage Lens: Analyze access patterns

Perfect! This is **exactly** the calibration I needed. Let me adjust the roadmap for you.

**Your Profile:**
- ✅ DevOps Engineer path (not Solutions Architect exam prep)
- ✅ Terraform (not CloudFormation)
- ✅ But still need architectural thinking (choosing right service, cost/performance trade-offs)
- ✅ Interview prep focus

This changes everything. Here's what I'll adjust:

---

## **🎯 ADJUSTED LEARNING APPROACH**

**What I'll KEEP:**
- Deep service knowledge (you need to know HOW services work)
- Architecture decisions (WHY to choose RDS vs DynamoDB)
- Troubleshooting scenarios (DevOps = firefighting)
- Cost optimization (DevOps = managing cloud spend)

**What I'll ADD:**
- **Terraform snippets** alongside console tasks
- **CI/CD integration** examples (how does this service fit in pipelines?)
- **Monitoring/alerting** focus (observability is DevOps bread and butter)
- **Infrastructure automation** patterns

**What I'll REDUCE:**
- Excessive console clicking (you'll automate this anyway)
- Solutions Architect exam trivia
- Deep dives into services you won't use daily (e.g., complex WAF rules)

---

## **📋 PHASE 3 CONTINUATION - DevOps Lens**

Before I continue with **RDS**, let me give you a **DevOps Reality Check** on S3:

### **S3 for DevOps Engineers - What Actually Matters:**

**In real DevOps work, you'll use S3 for:**

1. **Terraform State Backend** (most common)
```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/vpc/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

2. **Application Artifacts** (JAR, ZIP, Docker build contexts)
3. **Backup Storage** (database dumps, config backups)
4. **Log Aggregation** (before/instead of CloudWatch)
5. **Static Asset CDN** (S3 + CloudFront)

**What you DON'T need to obsess over:**
- Complex bucket policies (IaC handles this)
- Manual lifecycle policy creation (Terraform manages it)
- Deep ACL knowledge (deprecated, use policies)

**Your S3 knowledge level is GOOD ENOUGH for DevOps if you can:**
- ✅ Explain S3 consistency model (strongly consistent)
- ✅ Choose storage classes based on access patterns
- ✅ Secure buckets (encryption, block public access)
- ✅ Debug "access denied" errors
- ✅ Understand lifecycle policies conceptually
- ✅ Know when to use S3 vs EBS vs EFS

**Can you mentally answer these 3 questions?**
1. Why can't you use S3 as a database for transactional data?
2. If Terraform state is in S3, what happens if 2 people run `terraform apply` simultaneously?
3. Your CI/CD pipeline uploads build artifacts to S3. Should you use Standard or IA storage class?

If you can answer these quickly, **you're ready to move on.**

---

## **3.2 RDS - RELATIONAL DATABASE SERVICE**

**Project Integration:** Your news aggregator needs structured data storage:
- User accounts, preferences, subscriptions
- Article metadata (title, author, category, tags)
- Relational queries: "Show me all tech articles by Author X from last week"

**DevOps Angle:** RDS is where most production incidents happen. You need to know:
- Backup/restore procedures
- Performance tuning (why is the DB slow?)
- Failover mechanisms (what happens when primary DB dies?)
- Cost optimization (RDS is expensive)

---

### **Before Hands-On: The Big Decision**

**When do you use RDS vs DynamoDB?**

I'm going to give you a scenario, and you tell me which you'd choose:

**Scenario:**
Your news app has:
- 100,000 users
- Each user has: email, password, preferences
- Query patterns:
  - Login: Find user by email
  - Dashboard: Get user's subscribed categories
  - Settings: Update user preferences

**Quick poll (just think through it):**
- Would you use RDS (PostgreSQL) or DynamoDB?
- What factors matter most in this decision?

*(I'll give you the answer after the POC task, but thinking through it first will make the lesson stick)*

---

### **POC Task (Console):**

```
Goal: Launch a basic RDS instance and connect to it

1. Create RDS Subnet Group:
   - Name: news-db-subnet-group
   - VPC: news-aggregator-vpc
   - Subnets: Select 2 PRIVATE database subnets (from Phase 1)
   - Why private? Database should NEVER be internet-accessible

2. Create Security Group for RDS:
   - Name: news-db-sg
   - VPC: news-aggregator-vpc
   - Inbound rules:
     * Type: MySQL/Aurora (3306)
     * Source: app-tier-sg (only app servers can connect)
   - Outbound: Default (all traffic)

3. Launch RDS Instance:
   - Engine: PostgreSQL 15.x (or MySQL 8.x - your choice)
   - Template: Free tier (for POC)
   - DB instance identifier: news-aggregator-db-poc
   - Master username: admin
   - Master password: [Strong password - save it!]
   
   - Instance configuration:
     * db.t3.micro (free tier)
   
   - Storage:
     * Type: gp3
     * Size: 20 GB
     * Autoscaling: Disable (for cost control)
   
   - Connectivity:
     * VPC: news-aggregator-vpc
     * Subnet group: news-db-subnet-group
     * Public access: NO
     * Security group: news-db-sg
   
   - Database authentication: Password
   - Initial database name: newsdb
   
   - Backup:
     * Retention: 7 days (default)
     * Backup window: Default
   
   - Monitoring:
     * Enhanced monitoring: Disable (saves cost)
   
   - Maintenance:
     * Auto minor version upgrade: Disable (DevOps controls upgrades)

4. Wait for Creation (5-10 minutes):
   - Status: Creating → Available
   - Note the Endpoint: news-aggregator-db-poc.xxxxxxxxx.us-east-1.rds.amazonaws.com

5. Connect from EC2 Instance:
   
   SSH into an EC2 in the same VPC:
   
   # Install PostgreSQL client
   sudo yum install -y postgresql15
   
   # Connect to RDS
   psql -h news-aggregator-db-poc.xxxxxxxxx.us-east-1.rds.amazonaws.com \
        -U admin -d newsdb
   
   # Enter password when prompted
   
   # Test queries
   newsdb=> CREATE TABLE articles (
     id SERIAL PRIMARY KEY,
     title VARCHAR(255),
     content TEXT,
     published_at TIMESTAMP
   );
   
   newsdb=> INSERT INTO articles (title, content, published_at) 
            VALUES ('First Article', 'Hello RDS', NOW());
   
   newsdb=> SELECT * FROM articles;

6. Explore RDS Console:
   - Monitoring: CPU, connections, storage
   - Logs & events: Error logs, slow query logs
   - Configuration: Parameter groups, option groups
   - Maintenance: Pending maintenance windows

7. Test Connection Failure:
   - Try to connect from your laptop (not in VPC) → Timeout
   - This proves RDS is properly isolated
```

**Troubleshooting Challenge:**
If your `psql` connection hangs or times out:
1. Check security group (does it allow traffic from EC2's SG?)
2. Check subnet route table (can EC2 reach the database subnet?)
3. Check NACL (is there a DENY rule?)

---

### **Answer to Earlier Question:**

**RDS vs DynamoDB for user accounts?**

**For this use case: RDS (PostgreSQL)**

**Why?**
- ✅ Structured data with relationships (users → subscriptions → categories)
- ✅ Complex queries ("Find users subscribed to 'Technology' AND 'Politics'")
- ✅ ACID transactions (when user signs up, create user + default preferences atomically)
- ✅ Familiar SQL for your team

**When DynamoDB would be better:**
- Need single-digit millisecond latency at scale (millions of requests/sec)
- Simple key-value access patterns ("Get user by ID")
- Unpredictable/spiky traffic (DynamoDB auto-scales seamlessly)
- Want serverless pricing (pay per request, not per hour)

**DevOps Rule of Thumb:**
- **RDS:** Start here for most apps (familiar, flexible)
- **DynamoDB:** Migrate when you hit RDS scaling limits or need extreme performance

---

### **Production Task (Console + Terraform):**

```
Goal: Production RDS with Multi-AZ, read replicas, backups, monitoring

1. Create Production RDS (Console):
   
   - Engine: PostgreSQL 15.x
   - Template: Production
   - DB instance identifier: news-aggregator-db-prod
   
   - Master username: dbadmin
   - Master password: [Use AWS Secrets Manager]
   
   - Instance configuration:
     * db.r6i.large (memory-optimized)
     * Why? Databases need RAM for caching
   
   - Storage:
     * Type: gp3
     * Size: 100 GB
     * Provisioned IOPS: 3000 (baseline for gp3)
     * Autoscaling: Enable, max 500 GB
     * Encryption: Enable (use default KMS key or custom)
   
   - Availability & durability:
     * Multi-AZ: ENABLE ✅ (critical for production)
     * Creates synchronous standby in different AZ
   
   - Connectivity:
     * VPC: news-aggregator-vpc
     * Subnet group: news-db-subnet-group-prod
     * Public access: NO
     * Security group: news-db-prod-sg
   
   - Database authentication:
     * Password authentication: Enable
     * IAM database authentication: Enable (for Lambda access)
   
   - Monitoring:
     * Enhanced monitoring: Enable (1-second granularity)
     * Performance Insights: Enable (7-day retention free)
   
   - Additional configuration:
     * Initial database name: newsdb
     * DB parameter group: Create custom
     * Backup retention: 30 days
     * Backup window: 03:00-04:00 UTC (low traffic time)
     * Maintenance window: Sun 04:00-05:00 UTC
     * Deletion protection: ENABLE

2. Create Custom Parameter Group:
   
   - Family: postgres15
   - Name: news-db-params-prod
   - Parameters to tune:
     * max_connections: 200 (default is 100)
     * shared_buffers: 256MB (25% of RAM for caching)
     * effective_cache_size: 768MB (estimate of OS cache)
     * work_mem: 4MB (per-query memory)
     * maintenance_work_mem: 64MB (for VACUUM, CREATE INDEX)
     * checkpoint_completion_target: 0.9
     * wal_buffers: 16MB
     * random_page_cost: 1.1 (for SSD)
     * effective_io_concurrency: 200
     * log_min_duration_statement: 1000 (log queries >1s)

3. Create Read Replica:
   
   - Source: news-aggregator-db-prod
   - Replica identifier: news-aggregator-db-read-01
   - Instance class: db.r6i.large (same as primary)
   - Region: Same region (for low latency)
   - Multi-AZ: Disable (replicas are for read scaling, not HA)
   - Public access: NO
   
   Use cases:
   - Analytics queries (don't impact production writes)
   - Reporting dashboards
   - Read-heavy features (article browsing)

4. Configure Automated Backups:
   
   Automated backups (already configured):
   - Daily snapshot during backup window
   - Transaction logs backed up every 5 minutes
   - Point-in-time recovery (PITR) to any second in retention period
   
   Manual snapshots:
   - Create snapshot: news-db-before-migration-2025-10-04
   - Share snapshot with DR account (cross-account disaster recovery)
   - Copy snapshot to us-west-2 (cross-region DR)

5. Set Up CloudWatch Alarms:
   
   a. High CPU:
      - Metric: CPUUtilization
      - Threshold: > 80% for 5 minutes
      - Action: SNS → PagerDuty
   
   b. Low Free Storage:
      - Metric: FreeStorageSpace
      - Threshold: < 10 GB
      - Action: SNS + Lambda (trigger storage scaling)
   
   c. High Database Connections:
      - Metric: DatabaseConnections
      - Threshold: > 180 (90% of max_connections)
      - Action: SNS → Ops team
   
   d. Replica Lag:
      - Metric: ReplicaLag
      - Threshold: > 30 seconds
      - Action: SNS → Investigate replication issues
   
   e. Read/Write Latency:
      - Metric: ReadLatency, WriteLatency
      - Threshold: > 50ms average
      - Action: SNS → Check slow queries

6. Enable Performance Insights:
   
   - Already enabled during creation
   - Retention: 7 days (free) or 731 days (paid)
   
   What it shows:
   - Top SQL queries by load
   - Wait events (what's blocking queries?)
   - Database load over time
   
   DevOps use: "The DB is slow!" → Performance Insights → Find culprit query

7. Configure Enhanced Monitoring:
   
   - Granularity: 1 second
   - Sends OS metrics to CloudWatch Logs
   
   Metrics you get:
   - OS-level CPU (vs RDS instance CPU)
   - Memory (free, cached, buffers)
   - Swap usage
   - Disk I/O
   - Network throughput
   
   Cost: ~$1.50/month per instance

8. Store Credentials in Secrets Manager:
   
   - Create secret: rds/news-aggregator-db-prod
   - Secret type: Credentials for RDS database
   - Username: dbadmin
   - Password: [Generated]
   - Encryption key: Default
   - Automatic rotation: Enable (every 30 days)
   
   Application retrieves credentials:
   aws secretsmanager get-secret-value \
     --secret-id rds/news-aggregator-db-prod \
     --query SecretString --output text

9. Implement Connection Pooling:
   
   Problem: Each Lambda creates DB connection → 1000 Lambdas = 1000 connections = RDS overload
   
   Solution: RDS Proxy
   - Create proxy: news-db-proxy
   - Target: news-aggregator-db-prod
   - Connection pooling: Enable
   - Max connections: 100 (RDS Proxy multiplexes)
   - Idle timeout: 1800 seconds
   - IAM authentication: Enable
   
   Lambda connects to proxy endpoint, not directly to RDS
   Cost: $0.015/hour per vCPU (~$11/month for 1 proxy)
```

---

### **Terraform Translation (DevOps Mode):**

Here's how you'd create the production RDS in Terraform:

```hcl
# Secrets Manager for DB password
resource "random_password" "db_password" {
  length  = 16
  special = true
}

resource "aws_secretsmanager_secret" "db_credentials" {
  name        = "rds/news-aggregator-db-prod"
  description = "RDS master credentials"
}

resource "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = aws_secretsmanager_secret.db_credentials.id
  secret_string = jsonencode({
    username = "dbadmin"
    password = random_password.db_password.result
    engine   = "postgres"
    host     = aws_db_instance.main.endpoint
  })
}

# RDS Subnet Group
resource "aws_db_subnet_group" "main" {
  name       = "news-db-subnet-group-prod"
  subnet_ids = [aws_subnet.db_private_a.id, aws_subnet.db_private_b.id]

  tags = {
    Name        = "News DB Subnet Group"
    Environment = "production"
  }
}

# Security Group
resource "aws_security_group" "rds" {
  name        = "news-db-prod-sg"
  description = "Allow inbound traffic to RDS from app tier"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app_tier.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Custom Parameter Group
resource "aws_db_parameter_group" "main" {
  name   = "news-db-params-prod"
  family = "postgres15"

  parameter {
    name  = "max_connections"
    value = "200"
  }

  parameter {
    name  = "shared_buffers"
    value = "262144" # 256MB in 8KB pages
  }

  parameter {
    name  = "log_min_duration_statement"
    value = "1000" # Log queries > 1s
  }

  # Add more parameters as needed
}

# Main RDS Instance
resource "aws_db_instance" "main" {
  identifier = "news-aggregator-db-prod"

  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.r6i.large"

  allocated_storage     = 100
  max_allocated_storage = 500
  storage_type          = "gp3"
  storage_encrypted     = true
  iops                  = 3000

  db_name  = "newsdb"
  username = "dbadmin"
  password = random_password.db_password.result

  multi_az               = true
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  parameter_group_name   = aws_db_parameter_group.main.name

  backup_retention_period = 30
  backup_window           = "03:00-04:00"
  maintenance_window      = "sun:04:00-sun:05:00"

  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
  monitoring_interval             = 60
  monitoring_role_arn             = aws_iam_role.rds_monitoring.arn

  performance_insights_enabled    = true
  performance_insights_retention_period = 7

  deletion_protection = true
  skip_final_snapshot = false
  final_snapshot_identifier = "news-db-final-snapshot-${formatdate("YYYY-MM-DD-hhmm", timestamp())}"

  iam_database_authentication_enabled = true

  tags = {
    Name        = "News Aggregator DB"
    Environment = "production"
    ManagedBy   = "Terraform"
  }
}

# Read Replica
resource "aws_db_instance" "read_replica" {
  identifier = "news-aggregator-db-read-01"

  replicate_source_db = aws_db_instance.main.id
  instance_class      = "db.r6i.large"

  publicly_accessible = false
  skip_final_snapshot = true

  performance_insights_enabled = true

  tags = {
    Name        = "News DB Read Replica"
    Environment = "production"
  }
}

# RDS Proxy (for connection pooling)
resource "aws_db_proxy" "main" {
  name                   = "news-db-proxy"
  engine_family          = "POSTGRESQL"
  auth {
    secret_arn = aws_secretsmanager_secret.db_credentials.arn
  }
  role_arn               = aws_iam_role.rds_proxy.arn
  vpc_subnet_ids         = [aws_subnet.db_private_a.id, aws_subnet.db_private_b.id]
  require_tls            = true

  tags = {
    Name = "News DB Proxy"
  }
}

resource "aws_db_proxy_default_target_group" "main" {
  db_proxy_name = aws_db_proxy.main.name

  connection_pool_config {
    max_connections_percent      = 100
    max_idle_connections_percent = 50
    connection_borrow_timeout    = 120
  }
}

resource "aws_db_proxy_target" "main" {
  db_instance_identifier = aws_db_instance.main.id
  db_proxy_name          = aws_db_proxy.main.name
  target_group_name      = aws_db_proxy_default_target_group.main.name
}

# CloudWatch Alarms
resource "aws_cloudwatch_metric_alarm" "rds_cpu" {
  alarm_name          = "rds-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/RDS"
  period              = "300"
  statistic           = "Average"
  threshold           = "80"
  alarm_description   = "RDS CPU > 80%"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    DBInstanceIdentifier = aws_db_instance.main.id
  }
}

# Outputs
output "rds_endpoint" {
  value = aws_db_instance.main.endpoint
}

output "rds_proxy_endpoint" {
  value = aws_db_proxy.main.endpoint
}

output "read_replica_endpoint" {
  value = aws_db_instance.read_replica.endpoint
}
```

---

**DevOps Question:** Looking at this Terraform code, what's the biggest operational risk you see? *(Hint: Think about state management and secrets)*

---

### **Expert Secrets (DevOps Edition):**

**Multi-AZ is NOT a Read Replica:**

```
Common confusion:

Multi-AZ:
- Synchronous replication (no lag)
- Standby is NOT accessible (can't read from it)
- Automatic failover (60-120 seconds)
- Same region, different AZ
- Use for: High availability

Read Replica:
- Asynchronous replication (seconds of lag)
- Readable (use for queries)
- Manual promotion to master
- Can be in different region
- Use for: Read scaling, disaster recovery
```

**The Hidden Cost of Multi-AZ:**

```
Single-AZ db.r6i.large: $0.288/hour = $207/month
Multi-AZ db.r6i.large: $0.576/hour = $414/month

You're paying DOUBLE for the standby instance.

When to use Multi-AZ:
✅ Production databases
✅ Apps with <1 hour RTO (Recovery Time Objective)
✅ Compliance requirements for HA

When to skip Multi-AZ:
❌ Dev/test environments
❌ Non-critical data
❌ Can tolerate 30-60 min downtime for restore from snapshot
```

**Backup Strategy Deep Dive:**

```
Automated vs Manual Snapshots:

Automated:
- Taken daily during backup window
- Retained for 1-35 days
- DELETED when you delete the RDS instance
- Point-in-time recovery included

Manual:
- Taken on-demand
- Retained FOREVER (until you delete)
- NOT deleted when RDS instance deleted
- Use for: Pre-migration, pre-upgrade, compliance

Best practice:
1. Automated: 30 days (rolling window)
2. Manual: Major milestones (before schema changes)
3. Cross-region copy: Monthly (disaster recovery)
```

**Storage Autoscaling Gotcha:**

```
Configuration:
- Initial: 100 GB
- Max: 500 GB
- Threshold: 10% free space

What happens:
1. Storage hits 90 GB used
2. RDS scales to 110 GB (adds 10 GB or 10%, whichever is greater)
3. Next trigger: 99 GB used
4. RDS scales to 121 GB
... continues until 500 GB max

Problem: You can't REDUCE storage size
Once you hit 500 GB, you're paying for it forever

Solution: Monitor growth, set appropriate max
```

**Connection Pooling Math:**

```
Without RDS Proxy:
- 100 concurrent Lambdas
- Each opens 1 connection
- RDS max_connections = 200
- Result: 100 connections consumed

With RDS Proxy:
- 100 concurrent Lambdas
- Each connects to proxy
- Proxy maintains 10 connections to RDS
- Result: 10 connections consumed

Benefit: 10x more Lambdas can run simultaneously
Cost: $11/month for proxy
```

**Performance Tuning Workflow:**

```
1. Enable Performance Insights
2. Identify slow queries
3. Check execution plan:
   EXPLAIN ANALYZE SELECT ...
4. Add indexes:
   CREATE INDEX idx_articles_published ON articles(published_at);
5. Monitor impact in Performance Insights
6. Rinse and repeat

Common issues:
- Missing indexes (sequential scans)
- N+1 queries (application-level fix)
- Lock contention (tune isolation levels)
- Insufficient shared_buffers (increase RAM cache)
```

**Failover Behavior:**

```
Multi-AZ failover triggers:
- Primary instance fails
- AZ outage
- Manual reboot with failover
- OS patching

What happens:
1. RDS detects failure (60-120 seconds)
2. DNS flips to standby endpoint
3. Standby promoted to primary
4. New standby created
5. Total downtime: 60-120 seconds

Application impact:
- In-flight transactions: LOST
- Established connections: DROPPED
- Applications must retry connections

Best practice: Implement retry logic with exponential backoff
```

---

### **Interview Questions (DevOps Focus):**

**Operational:**
1. "Walk me through your process for upgrading RDS from PostgreSQL 14 to 15 with minimal downtime."
2. "Your RDS instance ran out of disk space at 3 AM. What happened and how do you prevent it?"
3. "Describe your backup and restore testing process. How often do you test restores?"

**Troubleshooting:**
4. "The application is getting 'too many connections' errors. What are the possible causes and solutions?"
5. "Performance Insights shows high CPU but no specific slow query. What's your debugging process?"
6. "After a failover, the application is reporting errors. What happened and how do you fix it?"

**Architecture:**
7. "When would you choose RDS Multi-AZ vs RDS with manual snapshots vs Aurora?"
8. "Your read replica is lagging 5 minutes behind primary. What causes this and how do you resolve it?"
9. "How do you implement zero-downtime database schema migrations?"

**Cost:**
10. "Your RDS bill increased from $200 to $800/month. How do you investigate and optimize?"

**Security:**
11. "How do you rotate RDS credentials without application downtime?"
12. "Explain IAM database authentication. When would you use it vs traditional passwords?"

**Disaster Recovery:**
13. "Design a cross-region disaster recovery strategy for RDS. What's your RPO and RTO?"

---

**Checkpoint Question:**

You've now learned VPC, EC2, ASG, ALB, S3, and RDS. 

**Can you mentally architect this:**
"A user uploads an image → Store in S3 → Lambda resizes it → Store metadata in RDS → Serve via CloudFront"

If you can sketch this architecture and identify where each service fits, **you're ready for DynamoDB.**

# **3.3 DYNAMODB - SERVERLESS NoSQL DATABASE**

**Project Integration:** DynamoDB will handle:
- Real-time user activity tracking (article views, clicks)
- Session management (user authentication tokens)
- High-velocity data (trending articles counter)
- Metadata cache (frequently accessed article summaries)

---

## **🤔 Before We Start - Challenge Your Thinking**

You've just learned RDS. Now we're introducing DynamoDB. Let me test something...

**Scenario:**
Your news aggregator tracks "article views" - every time a user opens an article, you increment a counter.

**Expected load:**
- 10,000 articles
- 1 million views per day
- Peak: 500 requests/second

**Question 1:** Would you store this view counter in:
- **Option A:** RDS (add a `view_count` column to articles table)
- **Option B:** DynamoDB (separate table for view counts)

**Question 2:** What factors are you considering in your decision? List 3 things that matter.

**Question 3:** What would happen if you chose RDS and traffic spiked to 5,000 requests/second?

---

Think through these. I'm not asking you to answer out loud (unless you want to), but **actively engage with the problem** before I give you the solution.

This is the #1 skill interviewers test: **Can you justify your architectural choices?**

---

## **The Answer (and Why It Matters):**

**You should use DynamoDB for this.**

**Why?**

```
RDS approach:
UPDATE articles SET view_count = view_count + 1 WHERE id = 'article-123';

Problems:
1. Every increment = database write
2. 500 writes/sec = database bottleneck
3. Scaling requires bigger instance ($$$)
4. Lock contention on hot articles
5. Connection pool exhaustion

DynamoDB approach:
UpdateItem with atomic counter increment

Benefits:
1. Single-digit millisecond latency at ANY scale
2. Auto-scales to millions of requests/sec
3. Pay per request (no idle cost)
4. No connection pooling needed
5. Built-in atomic operations
```

**The Pattern:**
- **RDS:** Complex queries, relationships, transactions
- **DynamoDB:** Simple key-value access, high throughput, low latency

---

## **DynamoDB Mental Model - Critical Foundation**

Before any hands-on, you need to understand **how DynamoDB thinks differently from SQL databases.**

### **Core Concept Check:**

**In RDS:**
```sql
SELECT * FROM articles WHERE category = 'Technology' AND published_at > '2025-10-01';
```
This is easy. You can filter on ANY column.

**In DynamoDB:**
You CANNOT do this query efficiently unless you design your table specifically for it.

**Why?** DynamoDB has no concept of "flexible queries." You must know your access patterns BEFORE designing the table.

---

### **The Three Things You MUST Understand:**

**1. Partition Key (Hash Key):**
```
Think of it as: "Which server stores this item?"

Example:
Partition Key = article_id

article_id="article-1" → Server A
article_id="article-2" → Server B
article_id="article-3" → Server A

DynamoDB hashes the partition key to determine storage location.
```

**2. Sort Key (Range Key):**
```
Think of it as: "How items are sorted within a partition"

Example:
Partition Key = user_id
Sort Key = timestamp

user_id="user-1", timestamp="2025-10-01" → First item for user-1
user_id="user-1", timestamp="2025-10-02" → Second item for user-1
user_id="user-1", timestamp="2025-10-03" → Third item for user-1

Query: "Get all activities for user-1 between Oct 1-3"
This is efficient because items are co-located and sorted.
```

**3. The Golden Rule:**
```
You can ONLY efficiently query by:
- Partition key (exact match)
- Partition key + sort key (range)

Everything else requires:
- Scan (reads entire table - SLOW and EXPENSIVE)
- Secondary indexes (we'll cover this)
```

---

**Pop Quiz (mental check):**

Given this table:
```
Partition Key: user_id
Sort Key: article_id
Attributes: title, category, timestamp
```

**Which queries are efficient?**
1. Get all articles read by user-123
2. Get article article-456 read by user-123
3. Get all articles in "Technology" category
4. Get all articles read yesterday

**Answers:**
1. ✅ Efficient (query by partition key)
2. ✅ Efficient (query by partition key + sort key)
3. ❌ Requires scan or secondary index
4. ❌ Requires scan or secondary index

**Key Insight:** If you answered these correctly, you understand DynamoDB fundamentals. If not, re-read the section above before continuing.

---

## **POC Task (Console):**

```
Goal: Create a DynamoDB table and understand basic operations

1. Create Table:
   - Table name: user-activity-poc
   - Partition key: user_id (String)
   - Sort key: timestamp (Number)
   - Table settings: On-demand (pay per request)
   - Encryption: AWS owned key (default)

2. Load Sample Data:
   
   Use DynamoDB Console → Tables → user-activity-poc → Explore items → Create item
   
   Item 1:
   {
     "user_id": "user-001",
     "timestamp": 1728000000,
     "action": "view_article",
     "article_id": "article-123",
     "category": "Technology"
   }
   
   Item 2:
   {
     "user_id": "user-001",
     "timestamp": 1728000060,
     "action": "view_article",
     "article_id": "article-456",
     "category": "Politics"
   }
   
   Item 3:
   {
     "user_id": "user-002",
     "timestamp": 1728000120,
     "action": "view_article",
     "article_id": "article-123",
     "category": "Technology"
   }
   
   Create 10 items total (mix of users and timestamps)

3. Query Operations:
   
   Console → Explore items → Query
   
   Query 1: Get all activity for user-001
   - Partition key: user_id = "user-001"
   - Execute
   - Result: Returns all items for user-001, sorted by timestamp
   
   Query 2: Get activity for user-001 in time range
   - Partition key: user_id = "user-001"
   - Sort key condition: timestamp BETWEEN 1728000000 AND 1728000100
   - Execute
   - Result: Returns filtered items

4. Scan Operation (see the difference):
   
   Console → Scan
   
   Scan 1: Find all "Technology" articles
   - Filter: category = "Technology"
   - Execute
   - Note: DynamoDB reads ALL items, then filters (inefficient)

5. PartiQL (SQL-like queries):
   
   Console → PartiQL editor
   
   SELECT * FROM "user-activity-poc" WHERE user_id = 'user-001'
   
   Note: This is just a wrapper around Query operation

6. Monitor Capacity:
   
   Console → Metrics → Consumed read/write capacity units
   - Each operation consumes RCUs (Read Capacity Units) or WCUs (Write Capacity Units)
   - On-demand mode: You don't see this, AWS handles it automatically
```

**Observation Exercise:**
Notice how querying by `user_id` is instant, but scanning for `category = "Technology"` reads the entire table. This is the fundamental difference between DynamoDB and SQL databases.

---

## **Production Task (Console + Concepts):**

```
Goal: Production DynamoDB with GSI, LSI, streams, and optimization

1. Design Table Structure (Single-Table Design):
   
   Table name: news-aggregator-prod
   
   This ONE table will store multiple entity types using overloaded keys:
   
   Entity: User
   PK: USER#user-123
   SK: PROFILE
   Attributes: email, name, subscription_tier
   
   Entity: User Activity
   PK: USER#user-123
   SK: ACTIVITY#2025-10-04T10:30:00Z
   Attributes: action, article_id
   
   Entity: Article
   PK: ARTICLE#article-456
   SK: METADATA
   Attributes: title, category, author
   
   Entity: Article Views
   PK: ARTICLE#article-456
   SK: VIEWS#2025-10-04
   Attributes: view_count
   
   Why single-table design? 
   - Reduces costs (one table vs multiple)
   - Enables transactions across entity types
   - Industry best practice for DynamoDB

2. Create Table:
   
   - Table name: news-aggregator-prod
   - Partition key: PK (String)
   - Sort key: SK (String)
   - Table class: Standard
   - Capacity mode: On-demand (switch to Provisioned later for cost optimization)
   - Encryption: Customer managed key (KMS)
   - Point-in-time recovery: Enable
   - DynamoDB Streams: Enable (New and old images)
   - Tags: Environment=Production

3. Create Global Secondary Index (GSI):
   
   GSI 1: Query articles by category
   - Index name: category-published-index
   - Partition key: category (String)
   - Sort key: published_at (String)
   - Projected attributes: All
   
   Use case: "Get all Technology articles published last week"
   Query: 
     PK = "Technology"
     SK BETWEEN "2025-09-27" AND "2025-10-04"
   
   GSI 2: Query by author
   - Index name: author-published-index
   - Partition key: author (String)
   - Sort key: published_at (String)
   - Projected attributes: Keys only (cheaper, fetch full item separately if needed)
   
   GSI 3: Trending articles
   - Index name: views-index
   - Partition key: date (String) # e.g., "2025-10-04"
   - Sort key: view_count (Number)
   - Projected attributes: All
   
   Use case: "Get top 10 most-viewed articles today"
   Query:
     PK = "2025-10-04"
     Sort descending by view_count
     Limit = 10

4. Set Up DynamoDB Streams + Lambda:
   
   Purpose: Trigger actions when data changes
   
   Example flow:
   1. User views article → UpdateItem (increment view_count)
   2. DynamoDB Streams captures the change
   3. Lambda function triggered
   4. Lambda checks if view_count crossed threshold (e.g., 10,000 views)
   5. Lambda sends SNS notification: "Article went viral!"
   
   Configuration:
   - Stream view type: New and old images
   - Batch size: 100
   - Starting position: Latest
   - Lambda function: article-analytics-processor

5. Enable Time To Live (TTL):
   
   - TTL attribute: expiration_time
   - Value: Unix timestamp
   
   Use case: Auto-delete user sessions after 24 hours
   
   Item:
   {
     "PK": "SESSION#abc123",
     "SK": "METADATA",
     "user_id": "user-001",
     "created_at": 1728000000,
     "expiration_time": 1728086400  # 24 hours later
   }
   
   DynamoDB automatically deletes items when expiration_time < current_time
   
   Cost: FREE (TTL deletes don't consume WCUs)

6. Configure Auto Scaling (if using Provisioned mode):
   
   Read capacity:
   - Minimum: 5 RCUs
   - Maximum: 100 RCUs
   - Target utilization: 70%
   
   Write capacity:
   - Minimum: 5 WCUs
   - Maximum: 50 WCUs
   - Target utilization: 70%
   
   Auto-scaling policy:
   - Scale out: When utilization > 70% for 2 minutes
   - Scale in: When utilization < 70% for 15 minutes (conservative)

7. Set Up CloudWatch Alarms:
   
   a. Throttled Requests:
      - Metric: UserErrors (throttling)
      - Threshold: > 10 in 5 minutes
      - Action: SNS → Increase capacity or investigate hot partition
   
   b. Consumed Capacity:
      - Metric: ConsumedReadCapacityUnits
      - Threshold: > 80% of provisioned capacity
      - Action: SNS → Consider increasing capacity
   
   c. System Errors:
      - Metric: SystemErrors
      - Threshold: > 0
      - Action: SNS + PagerDuty → AWS infrastructure issue

8. Implement Backup Strategy:
   
   Automated backups:
   - Point-in-time recovery (PITR): Enabled
   - Restore to any point in last 35 days
   - Cost: $0.20 per GB-month
   
   On-demand backups:
   - Create backup: news-aggregator-prod-backup-2025-10-04
   - Retention: 7 years (compliance)
   - Cross-region copy: us-west-2
   
   AWS Backup integration:
   - Backup plan: Daily at 2 AM UTC
   - Retention: 30 days
   - Lifecycle: Move to cold storage after 7 days

9. Export to S3 (for analytics):
   
   - Export table snapshot to S3
   - Format: DynamoDB JSON or Amazon Ion
   - Destination: s3://news-analytics-data/dynamodb-exports/
   - Use case: Run Athena queries on historical data without impacting production table
   
   Cost: $0.10 per GB exported

10. Implement DAX (DynamoDB Accelerator) - Optional:
    
    - Cluster name: news-aggregator-dax
    - Node type: dax.t3.small
    - Cluster size: 3 nodes (HA)
    - TTL: 5 minutes
    
    Use case: Cache for ultra-low latency reads (microseconds)
    
    Application code change:
    # Before (direct DynamoDB)
    response = dynamodb.get_item(TableName='news-aggregator-prod', Key=...)
    
    # After (through DAX)
    response = dax_client.get_item(TableName='news-aggregator-prod', Key=...)
    
    Cost: ~$0.12/hour per node (~$260/month for 3 nodes)
```

---

## **Terraform Example (Critical Patterns):**

```hcl
# DynamoDB Table with GSI
resource "aws_dynamodb_table" "main" {
  name           = "news-aggregator-prod"
  billing_mode   = "PAY_PER_REQUEST"  # On-demand
  hash_key       = "PK"
  range_key      = "SK"

  attribute {
    name = "PK"
    type = "S"
  }

  attribute {
    name = "SK"
    type = "S"
  }

  attribute {
    name = "category"
    type = "S"
  }

  attribute {
    name = "published_at"
    type = "S"
  }

  attribute {
    name = "author"
    type = "S"
  }

  # GSI for category queries
  global_secondary_index {
    name            = "category-published-index"
    hash_key        = "category"
    range_key       = "published_at"
    projection_type = "ALL"
  }

  # GSI for author queries
  global_secondary_index {
    name            = "author-published-index"
    hash_key        = "author"
    range_key       = "published_at"
    projection_type = "KEYS_ONLY"  # Cost optimization
  }

  # Enable TTL
  ttl {
    attribute_name = "expiration_time"
    enabled        = true
  }

  # Enable Streams
  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"

  # Enable PITR
  point_in_time_recovery {
    enabled = true
  }

  # Encryption
  server_side_encryption {
    enabled     = true
    kms_key_arn = aws_kms_key.dynamodb.arn
  }

  tags = {
    Name        = "News Aggregator Table"
    Environment = "production"
  }
}

# Lambda for DynamoDB Streams processing
resource "aws_lambda_event_source_mapping" "dynamodb_stream" {
  event_source_arn  = aws_dynamodb_table.main.stream_arn
  function_name     = aws_lambda_function.stream_processor.arn
  starting_position = "LATEST"
  batch_size        = 100

  # Error handling
  maximum_retry_attempts = 3
  maximum_record_age_in_seconds = 86400

  # Filter for specific events only
  filter_criteria {
    filter {
      pattern = jsonencode({
        eventName = ["INSERT", "MODIFY"]
        dynamodb = {
          NewImage = {
            PK = {
              S = [{ prefix = "ARTICLE#" }]
            }
          }
        }
      })
    }
  }
}

# CloudWatch Alarms
resource "aws_cloudwatch_metric_alarm" "throttled_requests" {
  alarm_name          = "dynamodb-throttled-requests"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "1"
  metric_name         = "UserErrors"
  namespace           = "AWS/DynamoDB"
  period              = "300"
  statistic           = "Sum"
  threshold           = "10"
  alarm_description   = "DynamoDB throttling detected"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    TableName = aws_dynamodb_table.main.name
  }
}
```

---

Got it! No more Terraform examples. Let's continue with pure concepts, hands-on console work, and DevOps wisdom.

---

## **Expert Secrets (DevOps Edition) - Continued:**

### **1. The Hot Partition Problem:**

```
Problem:
Your table has 10,000 articles, but one article goes viral.
- article-123: 1,000,000 reads/day
- All other articles: 100 reads/day total

DynamoDB partitions data by hash(PK).
All reads for article-123 hit ONE partition → Throttling

Solution 1: Write Sharding
Instead of:
PK = "ARTICLE#article-123"

Use:
PK = "ARTICLE#article-123#" + random(0-9)

Creates 10 copies of the item across 10 partitions.
When reading, your app queries all 10 shards and aggregates.

Solution 2: DAX (DynamoDB Accelerator)
Cache hot items in-memory. Viral article served from cache, not DynamoDB.

Solution 3: Read Sharding with GSI
Create a GSI with a synthetic partition key that distributes load.
```

**DevOps Reality:** This is the #1 performance issue in production DynamoDB. Know it cold.

---

### **2. Capacity Modes - The Cost Trade-off:**

```
On-Demand Mode:
- Pay per request: $1.25 per million writes, $0.25 per million reads
- Auto-scales instantly
- No capacity planning needed

Example cost:
- 10 million writes/month: $12.50
- 50 million reads/month: $12.50
Total: $25/month

Provisioned Mode:
- Pay per hour for reserved capacity
- WCU: $0.00065/hour, RCU: $0.00013/hour
- Must plan capacity, or use auto-scaling

Example cost (same workload):
- 10M writes/month = 4 WCU average
- 50M reads/month = 20 RCU average
- Provisioned: (4 × $0.00065 + 20 × $0.00013) × 730 hours = $3.80/month

Savings: 85%!

When to use each:
On-Demand:
✅ Unpredictable traffic
✅ New applications
✅ Dev/test environments
✅ Spiky workloads (Black Friday sales)

Provisioned:
✅ Predictable traffic
✅ Cost-sensitive (>80% cheaper at scale)
✅ Production apps with steady load
```

**DevOps Pro Tip:** Start with On-Demand, monitor for 2 weeks, analyze CloudWatch metrics, switch to Provisioned with appropriate capacity.

---

### **3. RCU/WCU Calculation (Interview Favorite):**

```
Write Capacity Unit (WCU):
- 1 WCU = 1 write/second for item up to 1 KB
- Item is 3.5 KB → Requires 4 WCUs (round up)
- Transactional write: 2x cost (2 WCUs for 1 KB item)

Read Capacity Unit (RCU):
- 1 RCU = 1 strongly consistent read/second for item up to 4 KB
- 1 RCU = 2 eventually consistent reads/second for item up to 4 KB
- Item is 10 KB → Requires 3 RCUs (10/4 = 2.5, round up to 3)
- Transactional read: 2x cost

Example Calculation:
You need to support:
- 100 writes/second, average item size 2 KB
- 500 reads/second, average item size 6 KB, eventually consistent

WCU calculation:
- 2 KB item = 2 WCUs
- 100 writes/sec × 2 WCUs = 200 WCUs needed

RCU calculation:
- 6 KB item with eventually consistent read
- 6 KB / 4 KB = 1.5 → rounds to 2 RCUs per strongly consistent read
- Eventually consistent = divide by 2 → 1 RCU per read
- 500 reads/sec × 1 RCU = 500 RCUs needed

Provisioned cost:
- 200 WCUs × $0.00065 × 730 hours = $94.90/month
- 500 RCUs × $0.00013 × 730 hours = $47.45/month
Total: $142.35/month

On-Demand cost (same workload):
- 100 writes/sec × 2,592,000 sec/month = 259.2M writes × $1.25/million = $324
- 500 reads/sec × 2,592,000 sec/month = 1,296M reads × $0.25/million = $324
Total: $648/month

Provisioned saves 78%!
```

**Interview Question They'll Ask:** "You have a table with 1,000 items. A scan operation reads all items. Each item is 3 KB. How many RCUs does this consume?"

**Answer:** 
- 1,000 items × 3 KB = 3,000 KB total
- Strongly consistent scan: 3,000 KB / 4 KB = 750 RCUs consumed
- Eventually consistent scan: 750 / 2 = 375 RCUs consumed

---

### **4. GSI vs LSI - The Critical Difference:**

```
Global Secondary Index (GSI):
- Different partition key than base table
- Can be created/deleted ANYTIME
- Has its own RCU/WCU allocation
- Eventually consistent reads ONLY
- Can query across all partitions

Example:
Base Table: PK = user_id, SK = timestamp
GSI: PK = article_id, SK = timestamp

Use case: "Get all users who viewed article-123"

Local Secondary Index (LSI):
- SAME partition key as base table
- Different sort key
- MUST be created at table creation (cannot add later)
- Shares RCU/WCU with base table
- Supports strongly consistent reads
- Limited to 10 GB per partition key value
- Max 5 LSIs per table

Example:
Base Table: PK = user_id, SK = timestamp
LSI: PK = user_id, SK = article_id

Use case: "Get all activity for user-123, sorted by article_id instead of timestamp"

When to use:
GSI: 99% of the time (flexible, can add later)
LSI: Rarely (only when you need strong consistency on alternate sort key)
```

**DevOps Reality:** Most engineers never use LSI. Stick with GSI unless you have a specific need for strong consistency on an alternate sort key.

---

### **5. DynamoDB Streams - Event-Driven Architecture:**

```
What Streams Give You:
- Ordered record of item-level changes (INSERT, MODIFY, REMOVE)
- Retention: 24 hours
- Each shard: Up to 1,000 records/second or 1 MB/second

Stream View Types:
1. KEYS_ONLY: Just the key attributes
2. NEW_IMAGE: The item after change
3. OLD_IMAGE: The item before change
4. NEW_AND_OLD_IMAGES: Both (most common)

Real-World Use Cases:

Use Case 1: Replication
- Write to DynamoDB table in us-east-1
- Stream triggers Lambda
- Lambda writes to DynamoDB table in us-west-2
- Result: Multi-region replication

Use Case 2: Analytics Pipeline
- User views article → DynamoDB write
- Stream triggers Lambda
- Lambda sends event to Kinesis Firehose
- Firehose writes to S3
- Athena queries for analytics

Use Case 3: Materialized Views
- Article view_count updated in main table
- Stream triggers Lambda
- Lambda updates "trending_articles" table with top 100

Use Case 4: Audit Trail
- Any change to user profile
- Stream captures old and new values
- Lambda writes to audit log table
- Immutable record of all changes
```

**DevOps Pattern:**
```
DynamoDB → Streams → Lambda → SNS → Multiple subscribers

Example:
Article published:
- Subscriber 1: Send email to subscribers
- Subscriber 2: Update search index (OpenSearch)
- Subscriber 3: Invalidate CloudFront cache
- Subscriber 4: Post to social media

Decoupled, event-driven architecture.
```

---

### **6. Single-Table Design - Advanced Pattern:**

```
Traditional approach (multiple tables):
- Users table
- Articles table
- Comments table
- Subscriptions table

Single-table design (one table):
All entities in ONE table using composite keys

Item Types:

User:
PK: USER#user-123
SK: #METADATA
Attributes: {email, name, created_at}

User Subscription:
PK: USER#user-123
SK: SUB#Technology
Attributes: {subscribed_at}

Article:
PK: ARTICLE#article-456
SK: #METADATA
Attributes: {title, content, author, published_at}

Comment on Article:
PK: ARTICLE#article-456
SK: COMMENT#2025-10-04T10:30:00Z#user-123
Attributes: {text, user_id}

Access Patterns:

1. Get user profile:
   GetItem(PK="USER#user-123", SK="#METADATA")

2. Get user's subscriptions:
   Query(PK="USER#user-123", SK begins_with "SUB#")

3. Get article with all comments:
   Query(PK="ARTICLE#article-456")
   Returns: Article metadata + all comments (sorted by timestamp)

4. Get user's comments across all articles:
   GSI: PK=user_id, SK=timestamp
   Query(PK="user-123")

Benefits:
✅ Fewer tables to manage
✅ Related data co-located (one query instead of multiple)
✅ Transactions across entity types
✅ Lower cost

Trade-offs:
❌ Steeper learning curve
❌ Harder to understand for SQL-minded devs
❌ Requires upfront access pattern design
```

**When to use:**
- Serverless applications (Lambda + DynamoDB)
- High-scale systems (millions of requests/sec)
- Cost-sensitive (every table costs money)

**When to avoid:**
- Team unfamiliar with NoSQL patterns
- Constantly changing access patterns
- Ad-hoc querying needs (use RDS instead)

---

### **7. Conditional Writes - Preventing Race Conditions:**

```
Problem: Two users try to like the same article simultaneously

Bad approach:
1. User A reads: like_count = 100
2. User B reads: like_count = 100
3. User A writes: like_count = 101
4. User B writes: like_count = 101
Result: Lost update! Should be 102.

Good approach 1: Atomic counter
UpdateItem with ADD operation:
- Atomically increments like_count
- No read required
- DynamoDB handles concurrency

Good approach 2: Conditional write
UpdateItem with condition:
- Only update if like_count = 100
- If condition fails, retry

Good approach 3: Optimistic locking (version attribute)
Item: {article_id, like_count, version}
UpdateItem:
- Condition: version = 5
- Update: like_count = 101, version = 6
- If version changed (another writer updated), condition fails, retry
```

**DevOps Reality:** Race conditions are subtle bugs. Use atomic operations whenever possible.

---

### **8. Batch Operations - Performance Optimization:**

```
Bad: Individual operations in a loop
for item in items:
    dynamodb.put_item(Item=item)

Result: 100 items = 100 network calls = slow

Good: BatchWriteItem
dynamodb.batch_write_item(
    RequestItems={
        'my-table': [
            {'PutRequest': {'Item': item1}},
            {'PutRequest': {'Item': item2}},
            ...
        ]
    }
)

Result: 100 items = 4 network calls (max 25 items per batch) = fast

Batch Operations:
- BatchGetItem: Retrieve up to 100 items
- BatchWriteItem: Put/Delete up to 25 items
- TransactWriteItems: Up to 25 items with ACID guarantees
- TransactGetItems: Up to 25 items with snapshot isolation

Gotcha: Unprocessed items
If DynamoDB throttles, some items may not process.
Response includes UnprocessedItems/UnprocessedKeys.
You MUST retry these with exponential backoff.
```

**DevOps Pattern:**
```python
def batch_write_with_retry(items):
    batch = items
    while batch:
        response = dynamodb.batch_write_item(RequestItems={...})
        batch = response.get('UnprocessedItems', {}).get('my-table', [])
        if batch:
            time.sleep(2 ** retry_count)  # Exponential backoff
```

---

### **9. Cost Optimization Strategies:**

```
Strategy 1: Use eventually consistent reads
- Strongly consistent: 1 RCU per 4 KB
- Eventually consistent: 0.5 RCU per 4 KB
- Savings: 50%
- Trade-off: Reads may be slightly stale (milliseconds)

Strategy 2: GSI projection optimization
- Projected attributes: ALL → Expensive
- Projected attributes: KEYS_ONLY → 90% cheaper
- If you only need keys for filtering, use KEYS_ONLY

Strategy 3: Compress large items
- Before: 100 KB item = 100 WCUs per write
- After compression: 20 KB = 20 WCUs
- Savings: 80%
- Tool: gzip, zstd

Strategy 4: Archive old data with TTL
- Enable TTL on old items
- DynamoDB deletes automatically (FREE, no WCU cost)
- Archive to S3 before deletion (using Streams + Lambda)

Strategy 5: Switch to Provisioned mode
- If traffic is predictable, provisioned is 70-90% cheaper
- Use CloudWatch to right-size capacity

Strategy 6: On-demand to Provisioned workflow
1. Run on-demand for 2 weeks
2. CloudWatch: Average RCU = 50, Max RCU = 200
3. Provision: 60 RCU baseline + auto-scaling to 250
4. Save 80% vs on-demand

Strategy 7: Reserved capacity (advanced)
- Commit to 1 or 3 years of provisioned capacity
- Savings: Up to 76%
- Only for stable, long-term production workloads
```

---

### **10. Common DynamoDB Anti-Patterns:**

```
❌ Anti-Pattern 1: Using DynamoDB like SQL
Bad: Scan entire table, filter in application code
Good: Design table for your query patterns

❌ Anti-Pattern 2: Large items
DynamoDB limit: 400 KB per item
Bad: Store entire 10 MB article in DynamoDB
Good: Store article metadata in DynamoDB, content in S3

❌ Anti-Pattern 3: Hot partitions
Bad: All requests go to one partition key (e.g., "global_counter")
Good: Shard across multiple partition keys

❌ Anti-Pattern 4: Using GSI for filtering
Bad: Create GSI for every possible filter combination
Good: Use DynamoDB for primary queries, Elasticsearch/OpenSearch for complex filtering

❌ Anti-Pattern 5: Not handling throttling
Bad: Application crashes on throttling
Good: Implement exponential backoff retry logic

❌ Anti-Pattern 6: Ignoring cost metrics
Bad: "It works, ship it!"
Good: Monitor CloudWatch for consumed capacity, optimize before scaling

❌ Anti-Pattern 7: Using DynamoDB for everything
Bad: Force-fit relational data into DynamoDB
Good: Use RDS for complex relationships, DynamoDB for key-value access
```

---

## **Interview Questions (DevOps Focus):**

**Conceptual:**
1. "Explain partition keys and sort keys. How does DynamoDB physically store data based on these?"
2. "What's the difference between a GSI and LSI? When would you use each?"
3. "How does DynamoDB achieve single-digit millisecond latency at any scale?"

**Scenario-Based:**
4. "Your DynamoDB table is being throttled despite having sufficient provisioned capacity. What could be wrong?"
5. "You need to query articles by category and date. Design the table structure and indexes."
6. "Your application writes 1,000 items/second to DynamoDB, but you see throttling. Debug this."

**Cost Optimization:**
7. "Your DynamoDB bill is $2,000/month. Walk me through your cost optimization strategy."
8. "When would you choose on-demand vs provisioned capacity mode?"

**Operations:**
9. "How do you perform a zero-downtime migration from RDS to DynamoDB?"
10. "Your Lambda is processing DynamoDB Streams but falling behind. How do you troubleshoot?"
11. "Describe your backup and restore strategy for a critical DynamoDB table."

**Advanced:**
12. "Explain DynamoDB Streams vs Kinesis Streams. When would you use each?"
13. "How do you implement optimistic locking in DynamoDB?"
14. "Design a system to count real-time article views (1M views/min) in DynamoDB without hot partitions."

**Troubleshooting:**
15. "Users report stale data from your DynamoDB table. What's the likely cause?"
16. "A GetItem operation returns no data, but you know the item exists. What are possible causes?"
17. "Your GSI is consuming more capacity than your base table. Why might this happen?"

---

## **Real-World Challenge:**

**Build a URL shortener like bit.ly:**

**Requirements:**
1. Shorten URL: POST /shorten → Returns short code (e.g., "abc123")
2. Redirect: GET /abc123 → 301 redirect to original URL
3. Analytics: Track click count per short URL
4. Handle 10,000 requests/second

**Design questions to think through:**
- Table structure (partition key, sort key)?
- How do you generate unique short codes?
- How do you handle click counting without hot partitions?
- On-demand or provisioned capacity?
- Do you need GSIs?
- How do you prevent duplicate short codes?

This is a classic interview question. Think through it before looking up solutions.

---

## **3.4 ELASTICACHE - IN-MEMORY DATA STORE**

**Project Integration:** ElastiCache will handle:
- Session storage (user authentication tokens)
- Application-level caching (reduce RDS/DynamoDB load)
- Rate limiting (track API requests per user)
- Real-time leaderboards (trending articles)

**DevOps Angle:** ElastiCache is where you go when your database can't keep up. It's your performance escape hatch.

---

### **The Fundamental Question:**

**Why do you need a cache?**

Think about this scenario:
```
Your news app homepage shows:
- Top 10 trending articles
- User's personalized feed
- Total article count

Without cache:
- Every page load = 3 database queries
- 1,000 users/second = 3,000 queries/second
- Database CPU: 100%
- Page load time: 2 seconds

With cache:
- Cache homepage data for 60 seconds
- 1,000 users/second = 17 database queries/minute (update cache)
- Database CPU: 5%
- Page load time: 50ms
```

**Caching is the #1 performance optimization in web applications.**

---

### **Redis vs Memcached - The Choice:**

```
Redis:
✅ Data structures (strings, lists, sets, sorted sets, hashes)
✅ Persistence (can survive restarts)
✅ Pub/Sub messaging
✅ Atomic operations
✅ Transactions
✅ Lua scripting
✅ Replication (master-replica)
✅ Cluster mode (sharding)
❌ Slightly slower than Memcached for simple key-value

Memcached:
✅ Extremely fast for simple key-value
✅ Multithreaded (better CPU utilization)
✅ Simpler (less to learn)
❌ No persistence (restart = data loss)
❌ No replication
❌ No advanced data structures
❌ No pub/sub

DevOps Rule:
- Choose Redis: 95% of the time (more features, persistence)
- Choose Memcached: Only if you need absolute maximum throughput for simple caching
```

---

### **POC Task (Console):**

```
Goal: Launch Redis cluster and understand basic caching patterns

1. Create ElastiCache Subnet Group:
   - Name: news-cache-subnet-group
   - VPC: news-aggregator-vpc
   - Subnets: Select 2 PRIVATE subnets (cache should never be public)

2. Create Security Group:
   - Name: news-cache-sg
   - Inbound: Redis (6379) from app-tier-sg
   - Outbound: All traffic

3. Create Redis Cluster (Non-Cluster Mode):
   - Engine: Redis 7.x
   - Name: news-cache-poc
   - Node type: cache.t3.micro (free tier eligible)
   - Number of replicas: 0 (single node for POC)
   - Multi-AZ: Disabled
   - Subnet group: news-cache-subnet-group
   - Security group: news-cache-sg
   - Backup retention: 0 days (POC)
   - Encryption at-transit: Disabled (enable in prod)
   - Encryption at-rest: Disabled (enable in prod)

4. Wait for Creation (5-10 minutes):
   - Status: Creating → Available
   - Note Primary Endpoint: news-cache-poc.abc123.0001.use1.cache.amazonaws.com:6379

5. Connect from EC2 Instance:
   
   SSH into EC2 in same VPC:
   
   # Install Redis CLI
   sudo yum install -y gcc
   wget http://download.redis.io/redis-stable.tar.gz
   tar xvzf redis-stable.tar.gz
   cd redis-stable
   make
   sudo cp src/redis-cli /usr/local/bin/
   
   # Connect to ElastiCache
   redis-cli -h news-cache-poc.abc123.0001.use1.cache.amazonaws.com
   
   # Test commands
   127.0.0.1:6379> SET article:123:views 1000
   OK
   127.0.0.1:6379> GET article:123:views
   "1000"
   127.0.0.1:6379> INCR article:123:views
   (integer) 1001
   127.0.0.1:6379> EXPIRE article:123:views 3600
   (integer) 1
   127.0.0.1:6379> TTL article:123:views
   (integer) 3599

6. Test Common Patterns:
   
   Pattern 1: Cache-aside (lazy loading)
   # Application pseudo-code:
   value = cache.get("article:123")
   if value is None:
       value = database.query("SELECT * FROM articles WHERE id=123")
       cache.set("article:123", value, ttl=3600)
   return value
   
   Pattern 2: Write-through
   # On article update:
   database.update("UPDATE articles SET title='New' WHERE id=123")
   cache.set("article:123", new_value, ttl=3600)
   
   Pattern 3: Session storage
   redis-cli> HSET session:user-123 email "user@example.com" logged_in "true"
   redis-cli> HGETALL session:user-123

7. Test Data Structures:
   
   # Sorted Set (for leaderboards)
   ZADD trending:today 1523 "article-1" 2891 "article-2" 5234 "article-3"
   ZREVRANGE trending:today 0 9 WITHSCORES  # Top 10 articles
   
   # List (for recent activity)
   LPUSH user:123:recent "article-1" "article-2" "article-3"
   LRANGE user:123:recent 0 4  # Last 5 activities
   
   # Set (for tags)
   SADD article:123:tags "technology" "aws" "devops"
   SMEMBERS article:123:tags
```

**Observation:** Notice how fast Redis is. Sub-millisecond responses.

---

### **Production Task (Console):**

```
Goal: Production Redis with replication, automatic failover, and backups

1. Create Production Redis Cluster (Cluster Mode Disabled):
   
   Why "Cluster Mode Disabled"?
   - Simpler to use
   - Supports all Redis commands
   - Good for most use cases (< 1 TB data, < 250K ops/sec)
   
   - Engine: Redis 7.x
   - Name: news-cache-prod
   - Node type: cache.r7g.large (memory-optimized, Graviton)
   - Number of replicas: 2 (1 primary + 2 replicas across AZs)
   - Multi-AZ: ENABLE (automatic failover)
   - Subnet group: news-cache-subnet-group-prod
   - Security group: news-cache-prod-sg
   
   - Encryption:
     * At-rest: Enable (KMS)
     * In-transit: Enable (TLS)
   
   - Backup:
     * Retention: 7 days
     * Backup window: 03:00-05:00 UTC
   
   - Maintenance window: Sun 05:00-06:00 UTC
   - SNS notification topic: cache-events
   
   - Parameter group: default.redis7
   - Log delivery: Enable slow log to CloudWatch

2. Create Redis Cluster Mode (for massive scale):
   
   When you need:
   - > 1 TB of data
   - > 250K operations/second
   - Data sharded across multiple nodes
   
   - Cluster mode: Enabled
   - Number of shards: 3
   - Replicas per shard: 2
   - Total nodes: 9 (3 primaries + 6 replicas)
   
   Data distribution:
   - Shard 1: Hash slot 0-5461
   - Shard 2: Hash slot 5462-10922
   - Shard 3: Hash slot 10923-16383
   
   Redis hashes your key and assigns to a shard.

3. Configure CloudWatch Alarms:
   
   a. High CPU:
      - Metric: EngineCPUUtilization
      - Threshold: > 75%
      - Action: SNS → Scale up node type
   
   b. High Memory:
      - Metric: DatabaseMemoryUsagePercentage
      - Threshold: > 80%
      - Action: SNS → Implement eviction policy or scale up
   
   c. Evictions (cache full):
      - Metric: Evictions
      - Threshold: > 100 in 5 minutes
      - Action: SNS → Cache is undersized
   
   d. Replication Lag:
      - Metric: ReplicationLag
      - Threshold: > 10 seconds
      - Action: SNS → Network or load issue
   
   e. Cache Hits vs Misses:
      - Metric: CacheHitRate
      - Threshold: < 80%
      - Action: Review caching strategy

4. Configure Eviction Policy:
   
   When cache is full, what to delete?
   
   Policies:
   - volatile-lru: Remove least recently used keys with expiration set
   - allkeys-lru: Remove least recently used keys (most common)
   - volatile-lfu: Remove least frequently used keys with expiration
   - allkeys-lfu: Remove least frequently used keys
   - volatile-random: Remove random keys with expiration
   - allkeys-random: Remove random keys
   - volatile-ttl: Remove keys with shortest TTL
   - noeviction: Return errors when memory full (not recommended)
   
   Recommendation: allkeys-lru for general caching

5. Implement Connection Pooling:
   
   Problem: Each Lambda/EC2 creates connection → 1,000 instances = 1,000 connections
   
   Solution: Use connection pool in your application
   
   Python example:
   import redis
   pool = redis.ConnectionPool(
       host='news-cache-prod.endpoint',
       port=6379,
       max_connections=50,
       socket_timeout=5,
       socket_connect_timeout=5
   )
   r = redis.Redis(connection_pool=pool)

6. Set Up Backup Strategy:
   
   Automated backups:
   - Daily snapshot during backup window
   - Retention: 7 days
   - Cost: $0.085 per GB-month
   
   Manual backups:
   - Before major deployments
   - Before Redis version upgrades
   - Retention: Until manually deleted
   
   Restore:
   - Create new cluster from snapshot
   - Update application endpoint
   - Cutover traffic

7. Enable Slow Log:
   
   - Parameter: slowlog-log-slower-than = 10000 (microseconds)
   - Tracks commands taking >10ms
   - View in CloudWatch Logs
   - Use for: Identifying inefficient cache operations
```

---

**Key concept check before I continue with expert secrets:**

**Can you explain the difference between these three Redis deployment options?**
1. Single node (no replicas)
2. Cluster mode disabled with replicas
3. Cluster mode enabled

Think about: availability, scalability, failover behavior, and use cases for each.

# **ELASTICACHE EXPERT SECRETS - Continued**

---

## **Answer to Deployment Options:**

Let me give you the answers since you're ready to move forward:

**1. Single Node (No Replicas):**
- **Availability:** None. Node dies = all data lost
- **Scalability:** Vertical only (upgrade node size)
- **Failover:** None
- **Use case:** Dev/test environments only

**2. Cluster Mode Disabled with Replicas:**
- **Availability:** High. Primary fails → replica auto-promoted in 60-90 seconds
- **Scalability:** Limited to single node's capacity (~700 GB)
- **Failover:** Automatic with Multi-AZ
- **Use case:** 95% of production workloads

**3. Cluster Mode Enabled:**
- **Availability:** Highest. Each shard has replicas
- **Scalability:** Horizontal (add shards, distribute data)
- **Failover:** Automatic per shard
- **Use case:** Massive scale (>1 TB, >250K ops/sec)

---

## **Expert Secrets (DevOps Critical):**

### **1. The Cache Stampede Problem:**

```
Scenario:
- Homepage cached for 1 hour
- 10,000 concurrent users
- Cache expires at exactly 12:00:00 PM

What happens:
12:00:00: Cache expires
12:00:01: 10,000 requests hit simultaneously
         All see cache miss
         All query database
         Database CPU: 100% → crashes
         Users see errors

This is called "Cache Stampede" or "Thundering Herd"

Solution 1: Cache Warming (proactive)
- Don't wait for cache to expire
- Background job refreshes cache BEFORE expiration
- Cron job at 11:55 AM regenerates cache

Solution 2: Probabilistic Early Expiration
- TTL = 3600 seconds
- At 3300 seconds, start randomly refreshing
- Spreads the load over time

Solution 3: Lock-based Refresh
- First request to see expired cache acquires lock
- Other requests wait or serve stale data
- Only one request hits database

Redis implementation:
SET cache_lock "locked" EX 10 NX
If successful, this request refreshes cache
If fails, another request is already refreshing
```

**DevOps Reality:** This causes 80% of cache-related production incidents. Know it.

---

### **2. Cache Invalidation Strategies:**

```
The Two Hard Problems in Computer Science:
1. Cache invalidation
2. Naming things
3. Off-by-one errors

Strategy 1: TTL-based (Time to Live)
SET article:123 "{...data...}" EX 3600
- Pros: Simple, automatic cleanup
- Cons: Stale data for up to TTL duration

Strategy 2: Write-through invalidation
On database update:
  database.update(...)
  cache.delete("article:123")
- Pros: Always fresh data
- Cons: Every write invalidates cache (reduces hit rate)

Strategy 3: Cache versioning
Key: article:123:v2
On update:
  Increment version
  New key: article:123:v3
- Pros: No explicit invalidation needed
- Cons: More memory usage (old versions linger until TTL)

Strategy 4: Pub/Sub invalidation (multi-instance)
Instance A updates database:
  database.update(...)
  redis.publish("invalidate", "article:123")

All instances subscribe:
  redis.subscribe("invalidate")
  On message: cache.delete(message)
  
- Pros: Works across multiple app servers
- Cons: Requires pub/sub infrastructure

Strategy 5: Tag-based invalidation
SET article:123 "{...}" 
SADD tag:technology:articles "article:123"
SADD tag:author:john:articles "article:123"

Invalidate all tech articles:
  members = SMEMBERS tag:technology:articles
  for article_id in members:
    DEL article_id

- Pros: Bulk invalidation
- Cons: Complex to maintain
```

**DevOps Best Practice:** Use TTL-based for most cases. Combine with write-through for critical data.

---

### **3. Cache Key Design Patterns:**

```
Bad Key Design:
- "user123"
- "data"
- "temp"

Problems:
- Collisions (multiple apps use same cluster)
- No structure
- Hard to debug

Good Key Design:
- "news:user:123:profile"
- "news:article:456:views"
- "news:session:abc-def-ghi"

Structure: {app}:{entity}:{id}:{attribute}

Benefits:
- Namespace isolation
- Easy to understand in logs
- Can use SCAN to find related keys
- Can set different TTLs by pattern

Advanced Pattern: Hierarchical keys
"news:prod:user:123:profile"
"news:staging:user:123:profile"

Allows same Redis cluster for multiple environments.
```

---

### **4. Memory Management:**

```
Redis Memory Usage:
- Actual data: ~70%
- Overhead: ~30% (metadata, pointers, fragmentation)

Example:
- cache.r7g.large: 13.07 GB RAM
- Usable for data: ~9 GB
- Store 1 million keys @ 9 KB each = 9 GB
- But overhead pushes you to ~12 GB used
- Result: Evictions start

Memory Optimization Tricks:

1. Use Hashes for Small Objects:
   Bad:
   SET user:1:name "John"
   SET user:1:email "john@example.com"
   SET user:1:age "30"
   Memory: ~300 bytes (3 keys + overhead)

   Good:
   HSET user:1 name "John" email "john@example.com" age "30"
   Memory: ~100 bytes (1 key + overhead)
   Savings: 67%

2. Use Compression:
   Before storing large JSON:
   import gzip
   compressed = gzip.compress(json_data.encode())
   redis.set(key, compressed)
   
   On retrieval:
   compressed = redis.get(key)
   json_data = gzip.decompress(compressed).decode()

3. Set Appropriate TTLs:
   Don't cache forever. Stale data = wasted memory.

4. Monitor Memory Fragmentation:
   INFO memory
   mem_fragmentation_ratio: 1.5
   
   If > 1.5, consider restart (releases fragmented memory)
```

---

### **5. Redis Persistence Options:**

```
RDB (Redis Database):
- Point-in-time snapshot
- Saved to disk every N minutes or M writes
- Fast restart (loads entire snapshot)
- Cons: Can lose up to N minutes of data

AOF (Append-Only File):
- Logs every write operation
- Three sync modes:
  * everysec (default): Fsync every second
  * always: Fsync every write (slow, safest)
  * no: Let OS decide (fastest, riskiest)
- Cons: Larger files, slower restart

ElastiCache Default: RDB snapshots
- Daily backups during backup window
- Snapshot on node replacement
- Restore creates new cluster from snapshot

Best Practice:
- Production: Use Multi-AZ (don't rely on persistence)
- Backups: For disaster recovery, not HA
- If you need durability: Consider DynamoDB instead
```

---

### **6. Connection Pool Sizing:**

```
Problem: How many connections should your app maintain to Redis?

Too few connections:
- Connection exhaustion
- Requests wait for available connection
- Increased latency

Too many connections:
- Redis overhead (each connection = memory)
- Wasted resources

Formula:
connections_per_instance = (peak_requests_per_second / avg_request_duration) * safety_margin

Example:
- Peak: 1,000 req/sec
- Avg duration: 5ms (0.005 sec)
- Safety margin: 2x

connections = (1000 / 200) * 2 = 10 connections per app instance

If you have 20 app instances:
Total connections to Redis = 20 * 10 = 200 connections

Redis limit (cache.r7g.large): 65,000 connections
You're safe.

Monitor:
CloudWatch → CurrConnections metric
If approaching limit, scale Redis node type.
```

---

### **7. Redis Commands Performance:**

```
Fast Commands (O(1)):
- GET, SET, INCR, DECR
- HGET, HSET
- LPUSH, RPUSH
- ZADD (single element)

Use freely, even at high volume.

Slow Commands (O(N)):
- KEYS * (scans entire keyspace)
- SMEMBERS (returns all set members)
- HGETALL (returns all hash fields)
- ZRANGE (returns range of sorted set)

Dangerous at scale:
KEYS pattern:*
If you have 10 million keys, this blocks Redis for seconds.

Alternative:
SCAN 0 MATCH pattern:* COUNT 100
Iterates in chunks, doesn't block.

Production Rule:
NEVER use KEYS in production.
Use SCAN for key discovery.

Benchmark tool:
redis-benchmark -h your-endpoint.cache.amazonaws.com -p 6379 -c 50 -n 100000
Tests throughput with 50 concurrent clients, 100K requests.
```

---

### **8. Multi-AZ Failover Behavior:**

```
Setup:
- Primary node in us-east-1a
- Replica node in us-east-1b
- Multi-AZ enabled

Failure Scenario:
1. Primary node crashes
2. ElastiCache detects failure (60 seconds)
3. DNS flips to replica endpoint
4. Replica promoted to primary
5. New replica launched in primary's AZ
6. Replication resumes

Total failover time: 1-2 minutes

Application Impact:
- In-flight commands: LOST
- Connections: DROPPED
- Applications must reconnect

Code Pattern:
try:
    result = redis.get(key)
except redis.ConnectionError:
    time.sleep(1)  # Wait for failover
    redis = reconnect()
    result = redis.get(key)

Monitoring:
CloudWatch event: "Failover Complete"
Set up SNS notification to alert ops team.
```

---

### **9. Cost Optimization:**

```
ElastiCache Pricing (us-east-1):
cache.t3.micro: $0.017/hour = $12/month
cache.r7g.large: $0.226/hour = $165/month
cache.r7g.xlarge: $0.452/hour = $330/month

Optimization Strategies:

1. Right-size node type:
   Monitor: DatabaseMemoryUsagePercentage
   If consistently < 50%, downsize
   If consistently > 80%, upsize

2. Reserved Instances:
   1-year commitment: 35% savings
   3-year commitment: 55% savings
   Only for stable production workloads

3. Use Graviton nodes (r7g):
   20% cheaper than Intel (r6i) for same performance

4. Reduce replica count in non-prod:
   Prod: 1 primary + 2 replicas (HA)
   Staging: 1 primary + 1 replica
   Dev: 1 primary + 0 replicas

5. Snapshot cost awareness:
   Snapshots: $0.085 per GB-month
   10 GB snapshot stored for 30 days = $0.85
   But 100 snapshots * 10 GB = $85/month
   Delete old snapshots!

6. Monitor data transfer:
   Cross-AZ traffic: $0.01/GB
   If replica in different AZ, you pay for replication traffic
   For dev/test, single-AZ is cheaper
```

---

## **Interview Questions (DevOps Focus):**

**Conceptual:**
1. "What's the difference between Redis and Memcached? When would you choose each?"
2. "Explain cache-aside vs write-through caching. What are the trade-offs?"
3. "What is cache stampede and how do you prevent it?"

**Architecture:**
4. "You need to cache user sessions. Would you use Redis or DynamoDB? Justify your choice."
5. "Design a caching layer for a news feed that updates every 5 minutes. Consider consistency, performance, and cost."
6. "Your Redis cluster is at 90% memory. What are your options?"

**Troubleshooting:**
7. "Application reports intermittent Redis connection errors. How do you debug this?"
8. "Cache hit rate dropped from 95% to 60%. Walk me through your investigation."
9. "After a Redis failover, the application crashes. What likely happened and how do you fix it?"

**Operations:**
10. "How do you perform a zero-downtime Redis version upgrade?"
11. "Your Redis backup is 50 GB. How long does a restore take and what's the process?"
12. "Describe your monitoring strategy for ElastiCache in production."

**Performance:**
13. "You're seeing 5ms latency on Redis GET operations. What could cause this and how do you optimize?"
14. "Your app has 100 instances, each with 20 Redis connections. Is this a problem?"

**Cost:**
15. "Redis bill is $1,000/month. Walk me through your cost optimization approach."

---

## **Real-World Challenge:**

**Build a rate limiter using Redis:**

**Requirements:**
- Limit each API key to 1,000 requests per hour
- Return 429 status code when limit exceeded
- Must be accurate (no overages)
- Must be fast (<5ms overhead)
- Must work across multiple API servers

**Design questions:**
- What Redis data structure would you use?
- What's the key naming scheme?
- How do you handle the sliding window?
- What happens if Redis is down?
- How do you monitor rate limit effectiveness?

This is a classic senior/staff engineer interview question. Think through the edge cases.

---

# **CHECKPOINT: You've Completed Storage & Databases**

You've now learned:
- ✅ S3 (object storage)
- ✅ RDS (relational databases)
- ✅ DynamoDB (NoSQL)
- ✅ ElastiCache (caching)

**Mental model check:** Can you answer these architectural questions?

1. **User uploads a profile photo. Where do you store it and why?**
   - S3, RDS, DynamoDB, or ElastiCache?

2. **You need to store user transaction history (millions of records). What do you use?**
   - Factors: query patterns, consistency needs, cost

3. **Homepage needs to show "Top 10 articles viewed in last hour". Design the data flow.**
   - Which services? Why? What's cached? What's real-time?

If you can articulate the trade-offs in these scenarios, **you're thinking like a senior engineer.**

---

# **PHASE 4: SERVERLESS & APPLICATION SERVICES** *(Week 8-10)*

*"Serverless doesn't mean no servers. It means no server management."*

This phase is CRITICAL for DevOps engineers. Serverless is where modern infrastructure is heading.

**Quick context check:** 
- Have you worked with AWS Lambda before (even a simple "Hello World")?
- Do you understand the concept of "event-driven architecture"?
- Are you familiar with the term "cold start"?

Your answers will help me calibrate the depth. But regardless, let's dive into **Lambda** - the heart of serverless computing.

---

## **4.1 AWS LAMBDA - SERVERLESS COMPUTE**

**Project Integration:** Lambda will:
- Process new articles when uploaded to S3 (extract metadata, generate summaries)
- Handle API requests via API Gateway (serverless REST API)
- Process DynamoDB Streams (update analytics, send notifications)
- Run scheduled tasks (fetch RSS feeds every hour)
- Resize images uploaded by users

**DevOps Angle:** Lambda is your automation engine. It's where you build event-driven systems that scale automatically.

---

### **Before Hands-On: The Paradigm Shift**

**Traditional (EC2):**
```
1. Provision EC2 instance
2. Install dependencies
3. Deploy code
4. Monitor CPU/memory
5. Scale manually or with ASG
6. Pay 24/7, even when idle
```

**Serverless (Lambda):**
```
1. Write function code
2. Upload to Lambda
3. AWS handles: servers, OS, scaling, patching, monitoring
4. Pay ONLY when function executes
5. Auto-scales from 0 to 10,000 concurrent executions
```

**The catch:** 
- Max execution time: 15 minutes
- Stateless (no persistent storage on function)
- Cold start latency (first invocation after idle)

**When to use Lambda:**
- ✅ Event-driven workloads (S3 upload, API request, schedule)
- ✅ Unpredictable or spiky traffic
- ✅ Short-lived tasks (<15 min)
- ✅ Glue code (connecting services)

**When NOT to use Lambda:**
- ❌ Long-running processes (>15 min)
- ❌ Stateful applications (use ECS/Fargate)
- ❌ Ultra-low latency (<10ms consistently needed)
- ❌ High-memory workloads (>10 GB RAM)

---

# **4.1 AWS LAMBDA - SERVERLESS COMPUTE** *(Continued)*

Perfect - I can see you're absorbing everything and ready to power through. Let me continue with Lambda, but I'm going to shift my approach slightly based on your learning style.

You've shown you can handle dense information and connect concepts yourself. So I'll give you the Lambda deep dive, but I want you to **actively engage** with it differently this time.

---

## **POC Task (Console) - But With a Twist**

I'm going to give you the steps, but **after each major section, I want you to pause and predict what will happen** before you do it. This builds intuition.

```
Goal: Create your first Lambda function triggered by S3

1. Create Lambda Function:
   - Go to Lambda console → Create function
   - Author from scratch
   - Function name: article-processor-poc
   - Runtime: Python 3.12
   - Architecture: x86_64 (we'll discuss arm64/Graviton later)
   - Execution role: Create new role with basic Lambda permissions
   - Create function

2. Examine What AWS Created:
   
   Look at the default code:
   
   import json
   
   def lambda_handler(event, context):
       return {
           'statusCode': 200,
           'body': json.dumps('Hello from Lambda!')
       }
   
   Three critical pieces:
   - lambda_handler: Entry point (AWS calls this function)
   - event: Input data (varies by trigger type)
   - context: Runtime information (request ID, memory, etc.)

3. Test the Function:
   
   - Click "Test"
   - Create test event: "test-event"
   - Use default template (hello-world)
   - Event JSON:
     {
       "key1": "value1",
       "key2": "value2"
     }
   
   - Click "Test"
   
   **STOP HERE - Predict:**
   What will the function return?
   How long will it take to execute?
   What appears in the logs?

4. Modify the Function (Parse Event Data):
   
   def lambda_handler(event, context):
       # Extract data from event
       key1 = event.get('key1', 'default')
       
       # Log to CloudWatch
       print(f"Received key1: {key1}")
       print(f"Request ID: {context.request_id}")
       print(f"Memory limit: {context.memory_limit_in_mb}MB")
       
       return {
           'statusCode': 200,
           'body': json.dumps({
               'message': f'Processed {key1}',
               'requestId': context.request_id
           })
       }
   
   - Deploy (Save)
   - Test again
   - Check CloudWatch Logs (Monitor → View logs in CloudWatch)

5. Add S3 Trigger:
   
   First, create S3 bucket:
   - news-lambda-test-[yourname]
   - Create folder: uploads/
   
   Add trigger to Lambda:
   - Add trigger → S3
   - Bucket: news-lambda-test-yourname
   - Event type: All object create events
   - Prefix: uploads/
   - Suffix: .json
   - Enable trigger
   
   **STOP - Predict:**
   What will the 'event' object contain when S3 triggers this function?
   How is it different from the test event you created?

6. Update Function to Process S3 Event:
   
   import json
   import boto3
   
   s3 = boto3.client('s3')
   
   def lambda_handler(event, context):
       # S3 event structure
       print("Full event:", json.dumps(event))
       
       # Extract S3 details
       bucket = event['Records'][0]['s3']['bucket']['name']
       key = event['Records'][0]['s3']['object']['key']
       
       print(f"Processing file: s3://{bucket}/{key}")
       
       # Read the file from S3
       response = s3.get_object(Bucket=bucket, Key=key)
       file_content = response['Body'].read().decode('utf-8')
       
       print(f"File content: {file_content}")
       
       return {
           'statusCode': 200,
           'body': json.dumps(f'Processed {key}')
       }
   
   Deploy and test:
   - Create test file: article.json
     {
       "title": "My First Article",
       "content": "Lambda is awesome!"
     }
   
   - Upload to S3: news-lambda-test-yourname/uploads/article.json
   - Check Lambda logs (should auto-trigger)

7. Permission Error? (You might see this):
   
   Error: Access Denied when reading from S3
   
   **STOP - Debug:**
   Why is this happening?
   What's missing?
   How do you fix it?
   
   (Hint: Remember IAM roles from Phase 1)
```

---

**Debugging Exercise Answer:**

The Lambda execution role needs S3 read permissions.

**Fix:**
```
1. Lambda → Configuration → Permissions
2. Click on execution role (opens IAM)
3. Attach policy: AmazonS3ReadOnlyAccess
   OR create inline policy:
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow",
       "Action": ["s3:GetObject"],
       "Resource": "arn:aws:s3:::news-lambda-test-yourname/uploads/*"
     }]
   }
4. Save
5. Re-upload file to S3 → Lambda triggers successfully
```

---

## **Key Observation Questions (Answer These Mentally):**

1. **When you uploaded to S3, how long did it take for Lambda to execute?**
   - This is your first taste of "cold start" vs "warm start"

2. **The event object from S3 - why is it structured as `event['Records'][0]`?**
   - Why an array? Why not just `event['s3']['bucket']`?

3. **What happens if you upload 10 files to S3 simultaneously?**
   - Does Lambda execute once or 10 times?
   - Are they serial or parallel?

Think through these. The answers reveal how Lambda fundamentally works.

---

## **The Answers (Critical Understanding):**

**1. Cold Start vs Warm Start:**
```
First invocation (cold start):
- AWS provisions container
- Loads runtime (Python)
- Loads your code
- Executes function
Time: 200ms - 2 seconds

Subsequent invocations (warm start):
- Container already exists
- Just executes function
Time: 1ms - 50ms

Container reuse window: ~5-15 minutes of idle time
After idle, next invocation is a cold start again.
```

**2. Event Records Array:**
```
S3 can batch multiple events into one Lambda invocation.
Example event:
{
  "Records": [
    { "s3": { "bucket": {...}, "object": {...} } },  # File 1
    { "s3": { "bucket": {...}, "object": {...} } }   # File 2
  ]
}

In practice, S3 usually sends 1 event per invocation.
But your code should handle multiple records.

Production pattern:
for record in event['Records']:
    bucket = record['s3']['bucket']['name']
    key = record['s3']['object']['key']
    process_file(bucket, key)
```

**3. Parallel Execution:**
```
10 files uploaded simultaneously = 10 Lambda invocations (parallel)

Lambda automatically scales:
- 0 to 1,000 concurrent executions (default account limit)
- Can request increase to 10,000+

Each invocation runs in isolated container.
No shared state between invocations (stateless).
```

---

## **Production Task (Console + Advanced Concepts):**

```
Goal: Production-grade Lambda with layers, environment variables, VPC, monitoring

1. Create Lambda Layer (for dependencies):
   
   Problem: Your function needs external libraries (requests, pillow, boto3)
   Bad approach: Package libraries with every function (large deployment)
   Good approach: Lambda Layer (shared across multiple functions)
   
   Create layer:
   a. Locally, create folder structure:
      python/
        lib/
          python3.12/
            site-packages/
              [your libraries]
   
   b. Install dependencies:
      pip install requests pillow -t python/lib/python3.12/site-packages/
   
   c. Zip the layer:
      zip -r layer.zip python/
   
   d. Upload to Lambda Layers:
      - Lambda console → Layers → Create layer
      - Name: news-dependencies-layer
      - Upload layer.zip
      - Compatible runtimes: Python 3.12
      - Create
   
   e. Attach to function:
      - Your Lambda → Configuration → Layers
      - Add layer → Custom layers → news-dependencies-layer

2. Configure Environment Variables:
   
   - Configuration → Environment variables
   - Add variables:
     * S3_BUCKET: news-processed-content
     * DYNAMODB_TABLE: news-aggregator-prod
     * CACHE_TTL: 3600
     * ENVIRONMENT: production
   
   Access in code:
   import os
   bucket = os.environ['S3_BUCKET']

3. Increase Memory and Timeout:
   
   - Configuration → General configuration → Edit
   - Memory: 1024 MB (default is 128 MB)
   - Timeout: 5 minutes (default is 3 seconds)
   - Ephemeral storage: 512 MB (default, max 10 GB)
   
   **Critical insight:**
   More memory = More CPU proportionally
   1024 MB gets ~0.6 vCPU
   3008 MB gets ~2 vCPU
   10240 MB gets full vCPU
   
   Sometimes, increasing memory REDUCES cost (faster execution)

4. Enable VPC Access (for RDS/ElastiCache):
   
   - Configuration → VPC
   - Select VPC: news-aggregator-vpc
   - Subnets: Select 2 PRIVATE subnets (app tier)
   - Security group: lambda-sg
     * Inbound: None needed (Lambda initiates connections)
     * Outbound: Allow to RDS (3306), ElastiCache (6379)
   
   **Warning:**
   VPC Lambda CANNOT access internet directly.
   Must route through NAT Gateway (costs $$$).
   
   Alternative: VPC Endpoints for AWS services (S3, DynamoDB, etc.)

5. Configure Concurrency Settings:
   
   - Configuration → Concurrency
   
   Reserved Concurrency:
   - Guarantees N executions available for this function
   - Prevents this function from consuming all account concurrency
   - Example: Reserve 100 for critical functions
   
   Provisioned Concurrency:
   - Pre-warmed containers (eliminates cold starts)
   - Costs: $0.0000041667 per GB-second
   - Use for: Latency-sensitive APIs
   
   Unreserved (default):
   - Shares account-level concurrency pool
   - Can burst to 1,000 concurrent (3,000 in some regions)

6. Set Up Dead Letter Queue (DLQ):
   
   - Configuration → Asynchronous invocation
   - DLQ: SNS topic or SQS queue
   
   Purpose: Failed invocations (after 2 retries) sent to DLQ for investigation
   
   Create SQS queue:
   - Name: lambda-failed-invocations
   - Type: Standard
   - Attach to Lambda DLQ configuration

7. Configure Retry Behavior:
   
   - Maximum age of event: 6 hours
   - Retry attempts: 2
   
   S3 triggers Lambda → Lambda fails → Retries 2 times → DLQ

8. Enable X-Ray Tracing:
   
   - Configuration → Monitoring and operations tools
   - Active tracing: Enable
   
   Adds X-Ray SDK to trace:
   - How long each AWS service call takes
   - Bottlenecks in your function
   - End-to-end request flow
   
   Access: X-Ray console → Service map

9. Set Up CloudWatch Alarms:
   
   a. High error rate:
      - Metric: Errors
      - Threshold: > 10 in 5 minutes
      - Action: SNS → PagerDuty
   
   b. High duration:
      - Metric: Duration
      - Threshold: > 4 minutes (80% of timeout)
      - Action: SNS → Investigate
   
   c. Throttling:
      - Metric: Throttles
      - Threshold: > 0
      - Action: SNS → Increase concurrency limit
   
   d. Cold starts (using custom metric):
      - Log cold start in function code
      - Create metric filter in CloudWatch Logs
      - Alarm if cold starts > 50% of invocations

10. Implement Function Code Best Practices:
    
    Production Lambda structure:
    
    import json
    import os
    import boto3
    from aws_xray_sdk.core import xray_recorder
    from aws_xray_sdk.core import patch_all
    
    # Patch AWS SDK for X-Ray
    patch_all()
    
    # Initialize clients OUTSIDE handler (reused across invocations)
    s3 = boto3.client('s3')
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table(os.environ['DYNAMODB_TABLE'])
    
    # Environment variables
    S3_BUCKET = os.environ['S3_BUCKET']
    ENVIRONMENT = os.environ.get('ENVIRONMENT', 'dev')
    
    @xray_recorder.capture('process_article')
    def process_article(bucket, key):
        """Process article from S3"""
        # Your business logic
        response = s3.get_object(Bucket=bucket, Key=key)
        content = response['Body'].read().decode('utf-8')
        article = json.loads(content)
        
        # Store in DynamoDB
        table.put_item(Item={
            'PK': f"ARTICLE#{article['id']}",
            'SK': 'METADATA',
            'title': article['title'],
            'processed_at': context.request_id
        })
        
        return article
    
    def lambda_handler(event, context):
        """Main handler"""
        print(f"Environment: {ENVIRONMENT}")
        print(f"Event: {json.dumps(event)}")
        
        try:
            # Process each S3 record
            for record in event['Records']:
                bucket = record['s3']['bucket']['name']
                key = record['s3']['object']['key']
                
                print(f"Processing: s3://{bucket}/{key}")
                article = process_article(bucket, key)
                print(f"Processed article: {article['id']}")
            
            return {
                'statusCode': 200,
                'body': json.dumps('Success')
            }
        
        except Exception as e:
            print(f"Error: {str(e)}")
            # Log error for monitoring
            raise  # Re-raise so Lambda retries
```

---

## **Critical Lambda Concepts (DevOps Must-Know):**

### **1. The Execution Model:**

```
Lambda lifecycle:

1. INIT phase (cold start only):
   - Download code
   - Start runtime
   - Run code OUTSIDE handler (imports, global variables)
   - Duration: 100ms - 2 seconds
   - Not billed (as of late 2024)

2. INVOKE phase:
   - Run handler function
   - Duration: Your function execution time
   - Billed per millisecond

3. Container reuse:
   - After invocation, container stays warm for 5-15 minutes
   - Next invocation within this window skips INIT
   - /tmp directory persists (can cache data between invocations)

Optimization:
# Bad (runs every invocation)
def lambda_handler(event, context):
    s3 = boto3.client('s3')  # Creates new client each time
    
# Good (runs once per container)
s3 = boto3.client('s3')  # Created during INIT
def lambda_handler(event, context):
    # Reuses existing client
```

### **2. Concurrent Execution Limits:**

```
Scenario: 1,000 requests/second, function takes 2 seconds

Concurrent executions needed = 1,000 * 2 = 2,000

Account limit (default): 1,000
Result: 1,000 throttled requests/second

Solutions:
1. Request limit increase (AWS Support)
2. Optimize function (reduce to 1 second = 1,000 concurrent)
3. Use SQS queue to buffer requests
4. Reserved concurrency for critical functions

Formula:
concurrent_executions = requests_per_second * average_duration_seconds
```

### **3. Lambda Pricing Breakdown:**

```
Two cost components:

1. Invocations: $0.20 per 1 million requests
2. Duration: $0.0000166667 per GB-second

Example calculation:
- 10 million invocations/month
- Average duration: 500ms
- Memory: 512 MB

Invocation cost:
10M * $0.20/1M = $2.00

Duration cost:
10M invocations * 0.5 seconds * 0.5 GB = 2.5M GB-seconds
2.5M * $0.0000166667 = $41.67

Total: $43.67/month

Free tier:
- 1M requests/month (forever)
- 400,000 GB-seconds/month (forever)

Cost optimization:
- Reduce memory if CPU-bound not memory-bound
- Optimize code (faster = cheaper)
- Use arm64 (Graviton) for 20% cost savings
```

### **4. Cold Start Deep Dive:**

```
What causes cold starts:
- First invocation ever
- Invocation after 15+ min idle
- Scaling up (new containers needed)
- Code/config changes

Cold start duration by runtime:
- Python: 150-400ms
- Node.js: 100-300ms
- Java: 500ms - 2 seconds (due to JVM)
- Go/Rust: 50-150ms (compiled languages)

Mitigation strategies:

1. Provisioned Concurrency:
   - Pre-warmed containers
   - Eliminates cold starts
   - Cost: ~$35/month per concurrent execution

2. Keep functions warm (cron):
   EventBridge rule every 5 minutes triggers Lambda
   Keeps container warm
   Cheaper than provisioned concurrency for low-traffic functions

3. Optimize package size:
   - Smaller deployment = faster cold start
   - Use Lambda Layers for dependencies
   - Remove unused code

4. Choose right runtime:
   - Python/Node for fast cold starts
   - Avoid Java unless necessary

5. Arm64 (Graviton):
   - 19% faster cold starts
   - 20% cheaper
   - Enable in configuration
```

### **5. VPC Lambda Performance:**

```
Old behavior (pre-2019):
- Lambda in VPC = slow cold start (10-30 seconds)
- Each function created ENI (Elastic Network Interface)
- ENI creation took 10+ seconds

New behavior (post-2019 improvement):
- Shared ENIs across functions
- Cold start overhead: <1 second
- VPC Lambda is now production-ready

But still consider:
- VPC Lambda needs NAT Gateway for internet ($$$)
- Or use VPC Endpoints for AWS services (free data transfer)

Pattern:
- Lambda needs S3/DynamoDB: NO VPC needed (public endpoints)
- Lambda needs RDS/ElastiCache: VPC required (private resources)
```

---

Before I continue with more Lambda patterns and then move to **API Gateway**, let me check:

**Quick self-assessment:**

1. Could you explain to another engineer why you'd initialize boto3 clients outside the handler function?

2. If someone said "my Lambda times out after 3 seconds," what's the first thing you'd check?

3. Your Lambda function costs $500/month. Without knowing what it does, what are the top 3 things you'd investigate to optimize cost?

Take 30 seconds to think through each. If you can answer these quickly, you're absorbing Lambda at the right depth. 


# **ADVANCED LAMBDA PATTERNS & API GATEWAY**

I see your style has updated - you want me to guide you through deeper understanding rather than just information dumps. Perfect timing, because Lambda and API Gateway is where **architectural thinking** becomes critical.

Let me shift approach here. Instead of giving you all the patterns, I want you to **solve real problems** and discover the patterns yourself.

---

## **🎯 LAMBDA CHALLENGE SERIES - Learn By Doing**

### **Challenge 1: The File Processing Pipeline**

**Scenario:**
Your news aggregator receives article submissions as Word documents (.docx) uploaded to S3. You need to:
1. Extract text from the Word document
2. Analyze sentiment (positive/negative/neutral)
3. Store results in DynamoDB
4. If sentiment is negative, send alert to SNS

**Your Task:**
Before I show you the solution, **design this architecture:**

**Questions for you to think through:**

1. **How many Lambda functions do you need?**
   - Option A: One function that does everything
   - Option B: Multiple functions, each with one responsibility
   - What are the trade-offs?

2. **The Word document processing requires a heavy library (python-docx, 50 MB).**
   - How does this affect cold start time?
   - Where should this library live?
   - What's the impact on your design?

3. **What happens if sentiment analysis fails but text extraction succeeded?**
   - Do you lose the extracted text?
   - How do you handle partial failures?

4. **The sentiment analysis takes 30 seconds. Text extraction takes 2 seconds.**
   - Does this change your architecture?
   - What about cost implications?

**Don't look ahead yet.** Spend 3-5 minutes thinking through this. Draw it on paper if that helps.

What would YOU design and why?

---

## **The Solution & Pattern Discovery:**

Since you said "continue," I'll give you the production-ready answer, but I want you to **compare it with what you were thinking.**

### **Optimal Architecture: Decoupled, Event-Driven**

```
S3 Upload (.docx)
    ↓ (S3 Event)
Lambda 1: Text Extractor (2 seconds, 1024 MB)
    ↓ (Writes to S3 as .txt)
S3 (.txt file)
    ↓ (S3 Event)
Lambda 2: Sentiment Analyzer (30 seconds, 512 MB)
    ↓ (Writes result to DynamoDB)
DynamoDB Stream
    ↓ (If sentiment = negative)
Lambda 3: Alert Handler (1 second, 128 MB)
    ↓
SNS Topic → Email/Slack
```

**Why this design?**

1. **Separation of Concerns:**
   - Each Lambda has ONE job (SRP - Single Responsibility Principle)
   - Easy to test, debug, and maintain
   - Can scale independently

2. **Resilience:**
   - If sentiment analysis fails, extracted text is already saved
   - Can retry sentiment analysis without re-extracting
   - Each step is idempotent (can run multiple times safely)

3. **Cost Optimization:**
   - Text extractor: High memory (needs library), short duration
   - Sentiment analyzer: Low memory, long duration
   - Alert handler: Minimal resources
   - Right-sized memory for each function

4. **Performance:**
   - Functions run in parallel if multiple uploads
   - No single function holds resources for 32 seconds
   - Faster time-to-first-result

**Alternative (Single Function) - Why It's Wrong:**
```
Problems:
- 32-second execution time
- High memory for entire duration (waste during sentiment analysis)
- Cold start loads 50 MB library even for simple tasks
- All-or-nothing: one step fails = entire pipeline fails
- Harder to debug ("where did it fail?")
```

---

### **Pattern Discovered: Lambda Orchestration**

**You just learned the fundamental pattern:**

**❌ Monolithic Lambda:**
```python
def lambda_handler(event, context):
    extract_text()      # 2s
    analyze_sentiment() # 30s
    send_alert()        # 1s
    return "Done"
```

**✅ Event-Driven Chain:**
```python
# Lambda 1
def extract_text_handler(event, context):
    text = extract_text(s3_file)
    s3.put_object(Bucket='processed', Key='text.txt', Body=text)
    # Automatically triggers Lambda 2 via S3 event

# Lambda 2  
def analyze_sentiment_handler(event, context):
    sentiment = analyze(text)
    dynamodb.put_item({'sentiment': sentiment})
    # Automatically triggers Lambda 3 via DynamoDB Stream

# Lambda 3
def alert_handler(event, context):
    if new_image['sentiment'] == 'negative':
        sns.publish(Message='Alert!')
```

---

## **🧠 Deep Concept Check:**

**Before we continue, answer this for yourself:**

You have a Lambda chain: Lambda A → Lambda B → Lambda C

**Question:** Lambda B fails after Lambda A succeeded. What happens to the data Lambda A processed?

**Think about:**
- Is it lost?
- Where is it stored?
- How does Lambda C know to skip processing?
- What if Lambda B fails 10 times?

This is **critical** for production systems. What's your answer?

---

### **The Answer: Event Sources & Destinations**

**Lambda has several invocation models:**

**1. Synchronous (Request-Response):**
```
Client → Lambda → Response

Examples:
- API Gateway → Lambda
- AWS CLI invoke
- Another Lambda calling this one

Retry behavior: Client's responsibility
Error handling: Client receives error
```

**2. Asynchronous (Fire-and-Forget):**
```
Event Source → Lambda (async)
              ↓
        Returns immediately
        
Examples:
- S3 events
- SNS notifications
- EventBridge rules

Retry behavior: Lambda retries 2 times automatically
Error handling: Can configure DLQ or destination
```

**3. Stream-Based (Poll-Based):**
```
Lambda polls source → Processes records

Examples:
- DynamoDB Streams
- Kinesis Streams
- SQS queues

Retry behavior: Lambda keeps trying until success or TTL expires
Error handling: Can configure failure destination
```

---

### **Lambda Destinations - The Modern Pattern**

**Old way (error handling):**
```python
def lambda_handler(event, context):
    try:
        process()
        # Manually send success to SNS
        sns.publish(TopicArn='success-topic', Message='Done')
    except Exception as e:
        # Manually send failure to SQS
        sqs.send_message(QueueUrl='dlq', Body=str(e))
```

**New way (Lambda Destinations):**
```
Configure in Lambda console:
- On Success: → SQS, SNS, Lambda, or EventBridge
- On Failure: → SQS, SNS, Lambda, or EventBridge

Your code:
def lambda_handler(event, context):
    process()  # If success, auto-sends to success destination
    # If exception, auto-sends to failure destination
```

**Why this matters:**
- Your code focuses on business logic
- AWS handles routing based on success/failure
- No manual try-catch for orchestration
- Cleaner, more maintainable code

---

## **Challenge 2: API Design Problem**

Now let's tackle **API Gateway** through problem-solving.

**Scenario:**
You're building a REST API for your news aggregator:
- `GET /articles` - List articles
- `GET /articles/{id}` - Get specific article
- `POST /articles` - Create article (authenticated users only)
- `DELETE /articles/{id}` - Delete article (admin only)

**Expected traffic:**
- 10,000 requests/second during peak
- Mostly GET requests (90%)
- Need to prevent abuse (rate limiting)
- Need to track API usage per customer

**Design Questions - Think Through These:**

1. **Where does authentication happen?**
   - Option A: Inside each Lambda function
   - Option B: At API Gateway level
   - What are the pros/cons?

2. **How do you handle the 10K requests/second?**
   - Each request triggers a Lambda
   - That's 10,000 concurrent Lambdas
   - Is this a problem?

3. **Rate limiting: "Each API key gets 1,000 requests/hour"**
   - Where do you implement this?
   - Lambda? API Gateway? ElastiCache?
   - What about distributed rate limiting (multiple API Gateway instances)?

4. **Caching `GET /articles` responses:**
   - Should this be cached?
   - Where? How long?
   - What invalidates the cache?

**Pause here.** These aren't rhetorical questions. I want you to think through the trade-offs.

If you were in an interview, how would you answer these?

---

## **The Production Architecture:**

```
Client Request
    ↓
CloudFront (Optional: Global CDN, caching)
    ↓
API Gateway
    ├─ Request Validation (schema check)
    ├─ Authentication (API Key / Cognito / Lambda Authorizer)
    ├─ Rate Limiting (Usage Plans)
    ├─ Response Caching (TTL: 300 seconds)
    ↓
Lambda (Business Logic)
    ↓
Backend (DynamoDB/RDS/S3)
```

**Let's unpack each decision:**

---

### **1. Authentication at API Gateway (Not Lambda)**

**Why:**
```
✅ Fails fast (rejected before Lambda invocation = no cost)
✅ Centralized (one place to manage auth)
✅ API Gateway handles token validation
✅ Lambda only receives authenticated requests

Three options:

Option A: API Keys (simplest)
- Client sends: x-api-key: abc123
- API Gateway validates against known keys
- Good for: Internal APIs, trusted clients

Option B: Cognito Authorizer
- Client sends: Authorization: Bearer <JWT>
- API Gateway validates with Cognito
- Good for: User-facing apps, OAuth flows

Option C: Lambda Authorizer (custom auth)
- API Gateway calls Lambda to validate token
- Lambda returns IAM policy (allow/deny)
- Good for: Custom auth logic (check database, external OAuth)
```

**Example Lambda Authorizer:**
```python
def lambda_handler(event, context):
    token = event['authorizationToken']
    
    # Validate token (check database, call auth service, etc.)
    if is_valid_token(token):
        user_id = extract_user_id(token)
        return generate_policy(user_id, 'Allow', event['methodArn'])
    else:
        return generate_policy('user', 'Deny', event['methodArn'])

def generate_policy(principal_id, effect, resource):
    return {
        'principalId': principal_id,
        'policyDocument': {
            'Version': '2012-10-17',
            'Statement': [{
                'Action': 'execute-api:Invoke',
                'Effect': effect,
                'Resource': resource
            }]
        },
        'context': {
            'userId': principal_id  # Passed to Lambda in event
        }
    }
```

**Critical:** Lambda Authorizer responses are **cached** for up to 1 hour. If you revoke a user's access, they can still call API for up to 1 hour (cache TTL).

---

### **2. Rate Limiting - Usage Plans**

**API Gateway has built-in rate limiting:**

```
Usage Plan Configuration:
- Throttle: 10,000 requests/second (burst: 5,000)
- Quota: 1,000,000 requests/month
- Per API Key: 1,000 requests/hour

How it works:
1. Client sends request with x-api-key header
2. API Gateway checks Usage Plan for this key
3. If within limits: Process request
4. If exceeded: Return 429 Too Many Requests

Cost: FREE (no extra charge for rate limiting)
```

**Implementation:**
```
1. Create API Key:
   - API Gateway → API Keys → Create
   - Key: abc123xyz

2. Create Usage Plan:
   - Name: Basic Plan
   - Throttle: 100 req/sec
   - Quota: 100,000 req/month
   - Associate with API Key

3. Attach to API Stage:
   - Link Usage Plan to "prod" stage
   - Now all requests require valid API key
```

**Advanced: Custom Rate Limiting (ElastiCache):**
```python
# When API Gateway rate limiting isn't enough
# (e.g., rate limit by user_id, not API key)

import redis
r = redis.Redis(host='cache-endpoint')

def lambda_handler(event, context):
    user_id = event['requestContext']['authorizer']['userId']
    key = f"rate_limit:{user_id}:{current_hour}"
    
    count = r.incr(key)
    r.expire(key, 3600)  # Expire after 1 hour
    
    if count > 1000:
        return {
            'statusCode': 429,
            'body': 'Rate limit exceeded'
        }
    
    # Process request
    return process_request(event)
```

---

### **3. Caching at API Gateway**

**Built-in response caching:**

```
Configuration:
- Cache capacity: 0.5 GB to 237 GB
- TTL: 0 to 3600 seconds
- Cache key: API path + query parameters

Example:
Request: GET /articles?category=tech
- First call: Lambda executes, response cached
- Subsequent calls: Served from cache (no Lambda invocation)
- Cache expires after TTL
- Cache invalidated on POST/PUT/DELETE

Cost: $0.020 per hour per GB (0.5 GB = $7.20/month)

When to use:
✅ Read-heavy APIs
✅ Data changes infrequently
✅ Same requests made repeatedly

When NOT to use:
❌ User-specific responses (every user has different data)
❌ Real-time data requirements
❌ Write-heavy APIs
```

**Cache key customization:**
```
By default: /articles?category=tech
Custom: Include Authorization header in cache key
         (each user gets separate cache entry)

Configure: API Gateway → Method → Integration Request → Cache key parameters
```

---

### **4. Handling 10K Requests/Second**

**Your question: "10,000 concurrent Lambdas - is this a problem?"**

**Short answer:** Maybe. Let's do the math.

**Calculation:**
```
Scenario 1: Fast Lambda (50ms)
- 10,000 req/sec
- 50ms per request = 0.05 seconds
- Concurrent executions = 10,000 * 0.05 = 500
- Account limit: 1,000 (default)
- Result: ✅ No problem

Scenario 2: Slow Lambda (2 seconds)
- 10,000 req/sec
- 2 seconds per request
- Concurrent executions = 10,000 * 2 = 20,000
- Account limit: 1,000 (default)
- Result: ❌ Throttling (19,000 requests rejected)

Solutions:
1. Optimize Lambda (reduce duration)
2. Request concurrency limit increase (AWS Support)
3. Implement caching (reduce Lambda invocations)
4. Use SQS queue to buffer requests
```

**Production pattern:**
```
High-traffic API:
1. CloudFront (global CDN, caches at edge)
2. API Gateway caching (reduces Lambda calls by 80%)
3. Lambda with reserved concurrency (guarantees capacity)
4. DynamoDB on-demand (scales automatically)

Result:
- 10,000 req/sec from users
- 9,000 served by CloudFront
- 900 served by API Gateway cache
- 100 hit Lambda
- Actual concurrency: 5 (if 50ms Lambda)
```

---

## **POC Task - API Gateway (Console):**

```
Goal: Create REST API with Lambda integration

1. Create Lambda Function:
   - Name: news-api-handler
   - Runtime: Python 3.12
   - Code:
   
   import json
   
   def lambda_handler(event, context):
       method = event['httpMethod']
       path = event['path']
       
       if method == 'GET' and path == '/articles':
           return {
               'statusCode': 200,
               'headers': {'Content-Type': 'application/json'},
               'body': json.dumps([
                   {'id': '1', 'title': 'Article 1'},
                   {'id': '2', 'title': 'Article 2'}
               ])
           }
       
       return {
           'statusCode': 404,
           'body': json.dumps({'message': 'Not found'})
       }

2. Create API Gateway (REST API):
   - API Gateway console → Create API
   - Choose: REST API (not HTTP API)
   - Protocol: REST
   - New API
   - Name: news-aggregator-api
   - Endpoint type: Regional

3. Create Resource and Method:
   - Create resource: /articles
   - Create method: GET
   - Integration type: Lambda Function
   - Lambda: news-api-handler
   - Use Lambda Proxy integration: ✅ (important!)

4. Deploy API:
   - Actions → Deploy API
   - Stage: dev
   - Note invoke URL: https://abc123.execute-api.us-east-1.amazonaws.com/dev

5. Test:
   curl https://abc123.execute-api.us-east-1.amazonaws.com/dev/articles
   
   Should return: [{"id": "1", "title": "Article 1"}, ...]
```

**Key concept: Lambda Proxy Integration**

```
Without proxy integration:
- API Gateway requires explicit mapping templates
- Must define input/output models
- Complex configuration

With proxy integration:
- Lambda receives full HTTP request in event object
- Lambda returns properly formatted HTTP response
- Much simpler (recommended)
```

---

**I'm going to pause here because we've covered a LOT.**

**Self-Assessment Questions:**

1. **If your Lambda takes 5 seconds and you're getting 1,000 req/sec, how many concurrent executions do you need?**

2. **Your API Gateway has caching enabled (300 second TTL). A user updates an article. How do you ensure GET /articles returns fresh data?**

3. **You want different rate limits for free users (100 req/hour) vs paid users (10,000 req/hour). How do you implement this?**

Take 2 minutes to think through these. They're not trick questions - they test if you understand the core concepts.



