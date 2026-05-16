# AWS Intern Task – Complete Step-by-Step Guide
## Highly Available Web Application on AWS

**Estimated Time:** 1.5 Days | **Level:** Beginner-Friendly | **By:** CitiusCloud Services

---

## 📅 DAY 1 PLAN (8–9 hours)

| Time | Task |
|------|------|
| Hour 1–2 | VPC, Subnets, IGW, NAT, Route Tables |
| Hour 2–3 | Security Groups |
| Hour 3–4 | EC2 Launch Template + NGINX Script |
| Hour 4–5 | Application Load Balancer + Target Group |
| Hour 5–6 | Auto Scaling Group |
| Hour 6–7 | Testing & Validation |

## 📅 DAY 2 PLAN (3–4 hours)

| Time | Task |
|------|------|
| Hour 1–2 | CloudFormation Template (automation) |
| Hour 2–3 | Documentation + Screenshots |
| Hour 3 | Teardown all resources |

---

## PART 1 – VPC & NETWORKING

### Step 1: Create a Custom VPC

1. Go to **AWS Console → VPC → Your VPCs → Create VPC**
2. Fill in:
   - **Name tag:** `intern-vpc`
   - **IPv4 CIDR block:** `10.0.0.0/16`
   - Leave everything else default
3. Click **Create VPC**

---

### Step 2: Create 4 Subnets (2 Public + 2 Private)

Go to **VPC → Subnets → Create subnet**

Create these 4 subnets one by one:

| Subnet Name | Type | Availability Zone | CIDR Block |
|---|---|---|---|
| `public-subnet-1` | Public | `ap-south-1a` | `10.0.1.0/24` |
| `public-subnet-2` | Public | `ap-south-1b` | `10.0.2.0/24` |
| `private-subnet-1` | Private | `ap-south-1a` | `10.0.3.0/24` |
| `private-subnet-2` | Private | `ap-south-1b` | `10.0.4.0/24` |

> ⚠️ Use `ap-south-1` (Mumbai) region since you're in India — it'll be faster and cheaper.

**Enable auto-assign public IP for public subnets:**
- Select `public-subnet-1` → Actions → Edit subnet settings
- Check **Enable auto-assign public IPv4 address** → Save
- Repeat for `public-subnet-2`

---

### Step 3: Create & Attach Internet Gateway

1. **VPC → Internet Gateways → Create internet gateway**
   - Name: `intern-igw`
   - Click Create
2. Select the IGW → **Actions → Attach to VPC**
   - Select `intern-vpc` → Attach

---

### Step 4: Create NAT Gateway

> The NAT Gateway lets private EC2 instances download packages (like NGINX) without being publicly accessible.

1. **VPC → NAT Gateways → Create NAT Gateway**
   - Name: `intern-nat`
   - Subnet: **`public-subnet-1`** ← MUST be a PUBLIC subnet
   - Connectivity type: **Public**
   - Click **Allocate Elastic IP** (this creates a public IP for the NAT)
   - Click Create NAT Gateway
2. Wait ~2 minutes for status to become **Available**

---

### Step 5: Configure Route Tables

#### 5a – Public Route Table
1. **VPC → Route Tables → Create route table**
   - Name: `public-rt`
   - VPC: `intern-vpc`
2. Select `public-rt` → **Routes tab → Edit routes → Add route**
   - Destination: `0.0.0.0/0`
   - Target: **Internet Gateway** → select `intern-igw`
   - Save
3. **Subnet associations tab → Edit subnet associations**
   - Select `public-subnet-1` and `public-subnet-2` → Save

#### 5b – Private Route Table
1. Create another route table: `private-rt` (same VPC)
2. **Routes → Edit routes → Add route**
   - Destination: `0.0.0.0/0`
   - Target: **NAT Gateway** → select `intern-nat`
   - Save
3. **Subnet associations → Edit**
   - Select `private-subnet-1` and `private-subnet-2` → Save

---

## PART 2 – SECURITY GROUPS

> Security groups act as virtual firewalls. We create TWO: one for the ALB, one for EC2.

### Step 6: Create ALB Security Group

1. **EC2 → Security Groups → Create security group**
   - Name: `alb-sg`
   - Description: `Allow HTTP from internet`
   - VPC: `intern-vpc`
2. **Inbound rules → Add rule:**
   - Type: `HTTP` | Protocol: `TCP` | Port: `80` | Source: `0.0.0.0/0`
3. Outbound: leave default (All traffic allowed)
4. Create security group

---

### Step 7: Create EC2 Security Group

1. Create another security group:
   - Name: `ec2-sg`
   - Description: `Allow HTTP from ALB only`
   - VPC: `intern-vpc`
2. **Inbound rules → Add rule:**
   - Type: `HTTP` | Protocol: `TCP` | Port: `80`
   - Source: **Custom** → search and select `alb-sg` (the security group ID)
3. **No SSH inbound rule** ← This proves private subnets are truly private
4. Create security group

> ✅ This is the least-privilege model — EC2 only accepts traffic FROM the ALB, not from the internet directly.

---

## PART 3 – EC2 LAUNCH TEMPLATE

### Step 8: Create Launch Template with NGINX

1. **EC2 → Launch Templates → Create launch template**
   - Name: `intern-lt`
   - Check: **Provide guidance to help me set up a template that I can use with EC2 Auto Scaling**

2. **AMI:** Amazon Linux 2023 (free tier eligible)

3. **Instance type:** `t2.micro` (free tier)

4. **Key pair:** No key pair ← (we're not SSHing in; this proves private subnet security)

5. **Network settings:**
   - Don't include subnet (ASG will handle this)
   - Security group: select `ec2-sg`

6. **Advanced details → User data** — paste this script:

```bash
#!/bin/bash
yum update -y
yum install -y nginx
systemctl start nginx
systemctl enable nginx

# Get instance metadata
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)

# Create custom webpage
cat > /usr/share/nginx/html/index.html <<EOF
<!DOCTYPE html>
<html>
<head>
  <title>AWS HA Web App</title>
  <style>
    body { font-family: Arial; background: #1a1a2e; color: white; 
           display: flex; justify-content: center; align-items: center; 
           height: 100vh; margin: 0; }
    .card { background: #16213e; padding: 40px; border-radius: 12px; 
            text-align: center; box-shadow: 0 0 30px rgba(0,150,255,0.3); }
    h1 { color: #00d4ff; }
    .info { background: #0f3460; padding: 15px; border-radius: 8px; margin: 10px 0; }
    .label { font-size: 12px; color: #aaa; }
    .value { font-size: 22px; font-weight: bold; }
  </style>
</head>
<body>
  <div class="card">
    <h1>🚀 CitiusCloud HA Web App</h1>
    <p>This page is served by an EC2 instance behind an ALB</p>
    <div class="info">
      <div class="label">Instance ID</div>
      <div class="value">$INSTANCE_ID</div>
    </div>
    <div class="info">
      <div class="label">Availability Zone</div>
      <div class="value">$AZ</div>
    </div>
    <p style="color:#00d4ff">Refresh to see load balancing in action! ↻</p>
  </div>
</body>
</html>
EOF
```

7. Click **Create launch template**

---

## PART 4 – APPLICATION LOAD BALANCER

### Step 9: Create Target Group

1. **EC2 → Target Groups → Create target group**
   - Target type: **Instances**
   - Name: `intern-tg`
   - Protocol: `HTTP` | Port: `80`
   - VPC: `intern-vpc`

2. **Health checks:**
   - Protocol: `HTTP`
   - Path: `/`
   - Healthy threshold: `2`
   - Unhealthy threshold: `2`
   - Interval: `30 seconds`

3. Click **Next → Create target group** (don't register targets manually — ASG will do it)

---

### Step 10: Create Application Load Balancer

1. **EC2 → Load Balancers → Create Load Balancer → Application Load Balancer**
   - Name: `intern-alb`
   - Scheme: **Internet-facing**
   - IP address type: **IPv4**

2. **Network mapping:**
   - VPC: `intern-vpc`
   - Mappings: check BOTH AZs
     - `ap-south-1a` → select `public-subnet-1`
     - `ap-south-1b` → select `public-subnet-2`

3. **Security groups:** remove default, add `alb-sg`

4. **Listeners and routing:**
   - Protocol: `HTTP` | Port: `80`
   - Default action: Forward to → `intern-tg`

5. Click **Create load balancer**
6. Copy the **DNS name** (you'll use this to test later)

---

## PART 5 – AUTO SCALING GROUP

### Step 11: Create Auto Scaling Group

1. **EC2 → Auto Scaling Groups → Create Auto Scaling group**
   - Name: `intern-asg`
   - Launch template: `intern-lt`
   - Click Next

2. **Network:**
   - VPC: `intern-vpc`
   - Subnets: select `private-subnet-1` AND `private-subnet-2`
   - Click Next

3. **Load balancing:**
   - Select **Attach to an existing load balancer**
   - Choose **intern-tg** from the dropdown
   - Enable **ELB health checks**
   - Click Next

4. **Group size:**
   - Desired: `2`
   - Minimum: `2`
   - Maximum: `4`
   - Click Next → Next → Create Auto Scaling group

5. Wait 2–3 minutes. Check **EC2 → Instances** — you should see 2 new instances launching.

---

## PART 6 – VALIDATION & TESTING

### ✅ Check 1: Load Balancing Works
1. Go to **EC2 → Load Balancers → intern-alb**
2. Copy the DNS name (looks like `intern-alb-xxxxx.ap-south-1.elb.amazonaws.com`)
3. Open it in a browser
4. **Refresh multiple times** — you should see different Instance IDs and AZs

### ✅ Check 2: Auto Scaling Self-Healing
1. Go to **EC2 → Instances**
2. Select one of the intern instances → **Instance State → Terminate**
3. Wait ~3 minutes
4. Refresh — ASG automatically launches a replacement instance

### ✅ Check 3: Private Subnets Are Private
- Try `ssh ec2-user@<private-ip>` from your laptop — it will **time out** ✓
- No inbound SSH rule + private subnet = no direct access

### ✅ Check 4: Least-Privilege Security Groups
1. Go to **EC2 → Security Groups → ec2-sg**
2. Check Inbound rules — it should show only port 80 from `alb-sg` (not from `0.0.0.0/0`)

---

## PART 7 – CLOUDFORMATION TEMPLATE

Save this as `intern-ha-webapp.yaml` and deploy via **CloudFormation → Create stack → Upload template**:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: CitiusCloud Intern Task - Highly Available Web Application

Parameters:
  EnvironmentName:
    Type: String
    Default: intern

Resources:

  # ─── VPC ───────────────────────────────────────────────
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags: [{Key: Name, Value: !Sub '${EnvironmentName}-vpc'}]

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags: [{Key: Name, Value: !Sub '${EnvironmentName}-igw'}]

  IGWAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  # ─── SUBNETS ───────────────────────────────────────────
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags: [{Key: Name, Value: !Sub '${EnvironmentName}-public-1'}]

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: !Select [1, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags: [{Key: Name, Value: !Sub '${EnvironmentName}-public-2'}]

  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.3.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags: [{Key: Name, Value: !Sub '${EnvironmentName}-private-1'}]

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.4.0/24
      AvailabilityZone: !Select [1, !GetAZs '']
      Tags: [{Key: Name, Value: !Sub '${EnvironmentName}-private-2'}]

  # ─── NAT GATEWAY ───────────────────────────────────────
  NATElasticIP:
    Type: AWS::EC2::EIP
    Properties: {Domain: vpc}

  NATGateway:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NATElasticIP.AllocationId
      SubnetId: !Ref PublicSubnet1
      Tags: [{Key: Name, Value: !Sub '${EnvironmentName}-nat'}]

  # ─── ROUTE TABLES ──────────────────────────────────────
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags: [{Key: Name, Value: !Sub '${EnvironmentName}-public-rt'}]

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: IGWAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicSubnet1RTAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties: {SubnetId: !Ref PublicSubnet1, RouteTableId: !Ref PublicRouteTable}

  PublicSubnet2RTAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties: {SubnetId: !Ref PublicSubnet2, RouteTableId: !Ref PublicRouteTable}

  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags: [{Key: Name, Value: !Sub '${EnvironmentName}-private-rt'}]

  PrivateRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NATGateway

  PrivateSubnet1RTAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties: {SubnetId: !Ref PrivateSubnet1, RouteTableId: !Ref PrivateRouteTable}

  PrivateSubnet2RTAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties: {SubnetId: !Ref PrivateSubnet2, RouteTableId: !Ref PrivateRouteTable}

  # ─── SECURITY GROUPS ───────────────────────────────────
  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP from internet
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
      Tags: [{Key: Name, Value: alb-sg}]

  EC2SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP from ALB only
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !Ref ALBSecurityGroup
      Tags: [{Key: Name, Value: ec2-sg}]

  # ─── ALB + TARGET GROUP ────────────────────────────────
  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: intern-tg
      Port: 80
      Protocol: HTTP
      VpcId: !Ref VPC
      HealthCheckPath: /
      HealthCheckIntervalSeconds: 30
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 2

  ALB:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: intern-alb
      Scheme: internet-facing
      Subnets: [!Ref PublicSubnet1, !Ref PublicSubnet2]
      SecurityGroups: [!Ref ALBSecurityGroup]

  ALBListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ALB
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref TargetGroup

  # ─── LAUNCH TEMPLATE ───────────────────────────────────
  LaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateName: intern-lt
      LaunchTemplateData:
        ImageId: ami-0f58b397bc5c1f2e8  # Amazon Linux 2023 ap-south-1
        InstanceType: t2.micro
        SecurityGroupIds: [!Ref EC2SecurityGroup]
        UserData:
          Fn::Base64: |
            #!/bin/bash
            yum update -y
            yum install -y nginx
            systemctl start nginx
            systemctl enable nginx
            INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
            AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)
            cat > /usr/share/nginx/html/index.html <<HTMLEOF
            <html><body style="background:#1a1a2e;color:white;text-align:center;padding:60px;font-family:Arial">
            <h1 style="color:#00d4ff">CitiusCloud HA Web App</h1>
            <p>Instance ID: <b>$INSTANCE_ID</b></p>
            <p>Availability Zone: <b>$AZ</b></p>
            <p>Refresh to see load balancing!</p>
            </body></html>
            HTMLEOF

  # ─── AUTO SCALING GROUP ────────────────────────────────
  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      AutoScalingGroupName: intern-asg
      LaunchTemplate:
        LaunchTemplateId: !Ref LaunchTemplate
        Version: !GetAtt LaunchTemplate.LatestVersionNumber
      MinSize: '2'
      MaxSize: '4'
      DesiredCapacity: '2'
      VPCZoneIdentifier: [!Ref PrivateSubnet1, !Ref PrivateSubnet2]
      TargetGroupARNs: [!Ref TargetGroup]
      HealthCheckType: ELB
      HealthCheckGracePeriod: 120

Outputs:
  ALBDNSName:
    Description: Load Balancer DNS - open this in browser
    Value: !GetAtt ALB.DNSName
```

---

## PART 8 – TEARDOWN CHECKLIST

> ⚠️ Delete resources in this EXACT ORDER to avoid errors. Missing steps = ongoing charges.

### Delete Order:

- [ ] **1. Auto Scaling Group** → EC2 → Auto Scaling Groups → Delete `intern-asg`
  - Wait for all instances to terminate
- [ ] **2. Application Load Balancer** → EC2 → Load Balancers → Delete `intern-alb`
- [ ] **3. Target Group** → EC2 → Target Groups → Delete `intern-tg`
- [ ] **4. Launch Template** → EC2 → Launch Templates → Delete `intern-lt`
- [ ] **5. NAT Gateway** → VPC → NAT Gateways → Delete `intern-nat`
  - ⏳ Wait ~5 minutes for NAT Gateway to fully delete
- [ ] **6. Release Elastic IP** → VPC → Elastic IPs → Release the address
- [ ] **7. Internet Gateway** → VPC → Internet Gateways → Detach from VPC, then Delete
- [ ] **8. Subnets** → VPC → Subnets → Delete all 4 subnets
- [ ] **9. Route Tables** → VPC → Route Tables → Delete `public-rt` and `private-rt`
- [ ] **10. Security Groups** → EC2 → Security Groups → Delete `ec2-sg` then `alb-sg`
- [ ] **11. VPC** → VPC → Your VPCs → Delete `intern-vpc`

### Verify No Charges:
- [ ] EC2 → Instances: No running instances
- [ ] VPC → NAT Gateways: None listed
- [ ] VPC → Elastic IPs: None allocated
- [ ] EC2 → Load Balancers: None listed

---

## 🏆 EVALUATION CHECKLIST

| # | Check | How to Prove |
|---|-------|--------------|
| 1 | Load balancing works | Screenshot of browser showing 2 different instance IDs on refresh |
| 2 | Auto Scaling self-heals | Screenshot of terminated instance + new instance launching |
| 3 | Private subnets are private | Screenshot of SSH timeout error |
| 4 | Least-privilege SG | Screenshot of ec2-sg showing source = alb-sg |
| 5 | Clean teardown | Screenshot of empty EC2 and VPC dashboards |

---

*Guide prepared for CitiusCloud AWS Intern Task | Mumbai Region (ap-south-1)*
