# AWS Infrastructure Implementation Example

**Complete Step-by-Step Guide: EC2, S3, Security Groups, and Multi-Service Deployment**

---

## Table of Contents

0. [Step-by-Step AWS Account & EC2 Creation](#0-step-by-step-aws-account--ec2-creation) ⭐ **START HERE**
1. [AWS EC2 Instance Configuration](#1-aws-ec2-instance-configuration)
2. [Security Groups & Networking](#2-security-groups--networking)
3. [Operating System Setup](#3-operating-system-setup)
4. [Directory Structure](#4-directory-structure)
5. [Database Configuration](#5-database-configuration)
6. [Service Port Mapping](#6-service-port-mapping)
7. [Systemd Service Management](#7-systemd-service-management)
8. [Environment Variables & API Keys](#8-environment-variables--api-keys)
9. [Process Management](#9-process-management)
10. [Backup Strategy](#10-backup-strategy)
11. [Monitoring & Health Checks](#11-monitoring--health-checks)

---

## 0. Step-by-Step AWS Account & EC2 Creation ⭐

**This section walks you through creating everything from scratch - no prior AWS experience needed.**

---

### Step 0.1: Create AWS Account

1. **Go to AWS Website**
   - Visit: https://aws.amazon.com
   - Click **"Create an AWS Account"** (top right)

2. **Enter Account Details**
   ```
   Root user email: your-email@example.com
   AWS account name: MyCompany-Production (or your preferred name)
   ```

3. **Contact Information**
   - Select **"Business"** or **"Personal"**
   - Fill in: Full name, Phone number, Country, Address

4. **Payment Information**
   - Add credit/debit card (required, but won't charge for free tier)
   - AWS will make a $1 verification charge (refunded immediately)

5. **Identity Verification**
   - Enter phone number
   - Receive verification code via SMS or voice call
   - Enter the 4-digit code

6. **Select Support Plan**
   - Choose **"Basic Support - Free"** (sufficient for most users)
   - Click **"Complete sign up"**

7. **Wait for Activation**
   - Account activation takes 5-10 minutes
   - You'll receive confirmation email when ready

---

### Step 0.2: Access AWS Console

1. **Sign In**
   - Go to: https://console.aws.amazon.com
   - Enter your root email and password
   - Click **"Sign in"**

2. **Enable MFA (Recommended)**
   - Click your account name (top right) → **"Security credentials"**
   - Under **"Multi-factor authentication (MFA)"**, click **"Assign MFA device"**
   - Choose **"Virtual MFA device"** (use Google Authenticator or similar)
   - Scan QR code with your phone
   - Enter two consecutive MFA codes
   - Click **"Assign MFA"**

3. **Select Region**
   - Top right corner: Select closest region
   - **Recommended regions:**
     - US East (N. Virginia): `us-east-1`
     - US West (Oregon): `us-west-2` ← **Recommended for this setup**
     - EU (Ireland): `eu-west-1`
   - **Important:** All resources in this guide will be created in your selected region

---

### Step 0.3: Create SSH Key Pair

**You need this to connect to your EC2 instance via SSH.**

1. **Navigate to EC2 Dashboard**
   - AWS Console → Search bar → Type **"EC2"** → Click **EC2**

2. **Create Key Pair**
   - Left sidebar → **"Network & Security"** → **"Key Pairs"**
   - Click **"Create key pair"** (orange button, top right)

3. **Key Pair Settings**
   ```
   Name: spyder-aws-key
   Key pair type: RSA
   Private key file format: .pem (for Mac/Linux) or .ppk (for Windows/PuTTY)
   Tags: (optional)
     Key: Environment
     Value: Production
   ```

4. **Download Key**
   - Click **"Create key pair"**
   - File downloads automatically: `spyder-aws-key.pem`
   - **CRITICAL:** This is your only chance to download this file!
   - Store it safely - you cannot re-download it later

5. **Set Key Permissions (Mac/Linux)**
   ```bash
   # Move key to SSH directory
   mv ~/Downloads/spyder-aws-key.pem ~/.ssh/
   
   # Set correct permissions (required for SSH)
   chmod 400 ~/.ssh/spyder-aws-key.pem
   ```

6. **Set Key Permissions (Windows)**
   - Right-click `spyder-aws-key.pem` → **Properties**
   - **Security** tab → **Advanced**
   - **Disable inheritance** → **Remove all inherited permissions**
   - **Add** → Type your username → **Check Names** → **OK**
   - Give **Read** permission only
   - **Apply** → **OK**

---

### Step 0.4: Create Security Group

**Security Groups act as a virtual firewall for your EC2 instance.**

1. **Navigate to Security Groups**
   - EC2 Dashboard → Left sidebar → **"Network & Security"** → **"Security Groups"**
   - Click **"Create security group"** (orange button, top right)

2. **Basic Details**
   ```
   Security group name: spyder-multi-service-sg
   Description: Security group for SPYDER multi-service platform (ports 22, 8046, 8055, 8056, 8057)
   VPC: (leave default - usually vpc-xxxxx)
   ```

3. **Add Inbound Rules**
   Click **"Add rule"** for each of these:

   **Rule 1 - SSH Access**
   ```
   Type: SSH
   Protocol: TCP
   Port range: 22
   Source: My IP (recommended) or Anywhere (0.0.0.0/0)
   Description: SSH access for administration
   ```

   **Rule 2 - SPYDER Main Service**
   ```
   Type: Custom TCP
   Protocol: TCP
   Port range: 8046
   Source: Anywhere-IPv4 (0.0.0.0/0)
   Description: SPYDER main research interface (curator1)
   ```

   **Rule 3 - Codex Standalone**
   ```
   Type: Custom TCP
   Protocol: TCP
   Port range: 8055
   Source: Anywhere-IPv4 (0.0.0.0/0)
   Description: Codex standalone AI service
   ```

   **Rule 4 - Codex V2 Orchestrator**
   ```
   Type: Custom TCP
   Protocol: TCP
   Port range: 8056
   Source: Anywhere-IPv4 (0.0.0.0/0)
   Description: Codex V2 universal orchestrator
   ```

   **Rule 5 - Book Catalog**
   ```
   Type: Custom TCP
   Protocol: TCP
   Port range: 8057
   Source: Anywhere-IPv4 (0.0.0.0/0)
   Description: Book catalog and management interface
   ```

4. **Outbound Rules**
   - Leave default (All traffic to 0.0.0.0/0)
   - This allows your instance to make outbound connections (API calls, downloads, etc.)

5. **Tags (Optional)**
   ```
   Key: Environment
   Value: Production
   
   Key: Project
   Value: SPYDER
   ```

6. **Create Security Group**
   - Click **"Create security group"** (orange button, bottom right)
   - Note down the Security Group ID (sg-xxxxxxxxx) - you'll need this later

---

### Step 0.5: Launch EC2 Instance (Detailed)

**This is the main step - follow carefully!**

1. **Navigate to EC2 Launch**
   - EC2 Dashboard → Click **"Launch instance"** (orange button, top right)

2. **Name and Tags**
   ```
   Name: spyder-production-server
   
   Additional tags (optional):
   - Environment: Production
   - Project: SPYDER
   - Owner: YourName
   ```

3. **Application and OS Images (AMI)**
   ```
   Quick Start: Ubuntu
   
   Select: Ubuntu Server 22.04 LTS (HVM), SSD Volume Type
   Architecture: 64-bit (x86)
   AMI ID: ami-xxxxxxxxx (changes by region)
   
   ✅ Free tier eligible (if applicable)
   ```

4. **Instance Type**
   ```
   Family: General purpose
   Type: t3.2xlarge
   
   Specifications:
   - vCPUs: 8
   - Memory: 32 GiB
   - Network Performance: Up to 5 Gigabit
   
   ⚠️ Not free tier eligible
   Cost: ~$0.33/hour = ~$240/month (us-west-2)
   ```

   **Alternative Budget Options:**
   ```
   t3.xlarge (4 vCPUs, 16 GB RAM): ~$120/month
   t3.large (2 vCPUs, 8 GB RAM): ~$60/month (minimum recommended)
   ```

5. **Key Pair**
   ```
   Select existing key pair: spyder-aws-key (created earlier)
   
   ✅ Check: "I acknowledge that I have access to the selected private key file"
   ```

6. **Network Settings**
   - Click **"Edit"** (right side)
   
   ```
   VPC: (default VPC is fine)
   Subnet: No preference (default subnet)
   Auto-assign public IP: Enable ✅
   
   Firewall (security groups):
   ◉ Select existing security group
   
   Select: spyder-multi-service-sg (created earlier)
   ```

7. **Configure Storage**
   ```
   Volume 1 (Root):
   - Size: 100 GiB (minimum)
   - Volume Type: gp3 (General Purpose SSD)
   - IOPS: 3000 (default)
   - Throughput: 125 MB/s (default)
   - Delete on termination: Yes ✅
   - Encrypted: Not encrypted (or enable if you prefer)
   ```

   **Optional - Add More Storage:**
   - Click **"Add new volume"** if you need separate data volume
   - Recommended: Keep everything on root volume for simplicity

8. **Advanced Details** (Expand this section)
   
   **Purchasing option:**
   ```
   ◉ On-Demand Instance (default)
   ☐ Spot instance (not recommended for production)
   ```

   **IAM instance profile:** (Optional - for S3 access)
   ```
   If you plan to use S3, create an IAM role with S3 access
   Otherwise: None
   ```

   **Shutdown behavior:**
   ```
   Stop (default) - recommended
   ```

   **Stop protection:**
   ```
   ☐ Enable (optional, but recommended for production)
   ```

   **Termination protection:**
   ```
   ☐ Enable ✅ (highly recommended to prevent accidental deletion)
   ```

   **User data:** (Leave empty - we'll configure manually)

9. **Review Summary**
   - Right panel shows your configuration
   - Estimated cost per month: ~$240
   - Review all settings carefully

10. **Launch Instance**
    - Click **"Launch instance"** (orange button, bottom right)
    - You'll see: **"Successfully initiated launch of instance i-xxxxxxxxx"**
    - Click **"View all instances"**

11. **Wait for Instance to Start**
    ```
    Instance State: pending → running (takes 1-2 minutes)
    Status checks: Initializing → 2/2 checks passed (takes 2-3 minutes)
    ```

12. **Note Important Information**
    ```
    Instance ID: i-0123456789abcdef0
    Public IPv4 address: 44.225.226.126 (example)
    Private IPv4 address: 172.31.44.26 (example)
    Instance state: running
    ```

---

### Step 0.6: Allocate and Associate Elastic IP

**Elastic IP provides a static public IP that doesn't change when you restart your instance.**

1. **Navigate to Elastic IPs**
   - EC2 Dashboard → Left sidebar → **"Network & Security"** → **"Elastic IPs"**

2. **Allocate Elastic IP**
   - Click **"Allocate Elastic IP address"** (orange button, top right)
   
   ```
   Network Border Group: (keep default)
   Public IPv4 address pool: Amazon's pool of IPv4 addresses
   
   Tags (optional):
   - Name: spyder-production-ip
   - Environment: Production
   ```

   - Click **"Allocate"**
   - Note the new IP address: `44.225.226.126` (example)

3. **Associate Elastic IP with Instance**
   - Select the newly allocated Elastic IP (checkbox)
   - **Actions** → **"Associate Elastic IP address"**
   
   ```
   Resource type: Instance
   Instance: i-xxxxxxxxx (your spyder instance)
   Private IP address: (auto-filled)
   ```

   - Click **"Associate"**

4. **Verify Association**
   - Go back to EC2 Instances
   - Your instance should now show the Elastic IP as its public IP
   - **This IP will never change** (even if you stop/start the instance)

---

### Step 0.7: Create S3 Bucket (Optional - for backups)

**S3 provides cheap, reliable storage for database backups.**

1. **Navigate to S3**
   - AWS Console → Search **"S3"** → Click **S3**

2. **Create Bucket**
   - Click **"Create bucket"** (orange button)

3. **Bucket Settings**
   ```
   Bucket name: spyder-database-backups-YOURNAME
   (Must be globally unique - add your name/company)
   
   AWS Region: us-west-2 (or your EC2 region - MUST MATCH!)
   ```

4. **Object Ownership**
   ```
   ◉ ACLs disabled (recommended)
   ```

5. **Block Public Access**
   ```
   ✅ Block all public access (keep checked)
   ```

6. **Bucket Versioning**
   ```
   ☐ Disable (or enable if you want version history)
   ```

7. **Default Encryption**
   ```
   Encryption type:
   ◉ Server-side encryption with Amazon S3 managed keys (SSE-S3)
   Bucket Key: Enable ✅
   ```

8. **Advanced Settings** (optional)
   ```
   Object Lock: Disable (not needed)
   ```

9. **Create Bucket**
   - Click **"Create bucket"** (orange button, bottom)
   - Your bucket is now ready: `s3://spyder-database-backups-yourname/`

10. **Set Lifecycle Policy** (Cost Optimization)
    - Click on your bucket name
    - Go to **"Management"** tab
    - Click **"Create lifecycle rule"**
    
    ```
    Rule name: archive-old-backups
    
    Rule scope: ◉ Apply to all objects
    
    Lifecycle rule actions:
    ✅ Transition current versions of objects between storage classes
    ✅ Expire current versions of objects
    
    Transitions:
    - Move to Glacier after: 30 days
    
    Expiration:
    - Delete after: 90 days
    ```

    This will:
    - Keep backups in standard S3 for 30 days (~$0.023/GB/month)
    - Move to Glacier for days 30-90 (~$0.004/GB/month)
    - Delete after 90 days
    - **Saves ~70% on storage costs!**

---

### Step 0.8: Connect to Your EC2 Instance

**Time to access your new server via SSH!**

#### For Mac/Linux:

1. **Open Terminal**

2. **Connect via SSH**
   ```bash
   ssh -i ~/.ssh/spyder-aws-key.pem ubuntu@44.225.226.126
   ```
   Replace `44.225.226.126` with your Elastic IP

3. **First Connection**
   - You'll see: `The authenticity of host... can't be established`
   - Type **"yes"** and press Enter

4. **You're In!**
   ```
   Welcome to Ubuntu 22.04.5 LTS
   ubuntu@ip-172-31-44-26:~$
   ```

#### For Windows (Using PuTTY):

1. **Download PuTTY**
   - Go to: https://www.putty.org/
   - Download and install PuTTY

2. **Convert .pem to .ppk**
   - Open **PuTTYgen** (installed with PuTTY)
   - Click **"Load"**
   - Select your `.pem` file
   - Click **"Save private key"**
   - Save as `spyder-aws-key.ppk`

3. **Connect with PuTTY**
   - Open **PuTTY**
   - **Host Name:** `ubuntu@44.225.226.126`
   - **Port:** 22
   - **Connection type:** SSH
   - Left panel: **Connection** → **SSH** → **Auth** → **Credentials**
   - **Private key file:** Browse and select `spyder-aws-key.ppk`
   - Go back to **Session** → **Saved Sessions:** Type "SPYDER" → Click **Save**
   - Click **"Open"**

4. **Accept Security Alert**
   - Click **"Accept"** on first connection

5. **You're Connected!**
   ```
   ubuntu@ip-172-31-44-26:~$
   ```

---

### Step 0.9: Initial Server Configuration

**Run these commands immediately after first login:**

```bash
# 1. Update system packages
sudo apt update && sudo apt upgrade -y

# 2. Install essential tools
sudo apt install -y \
    python3-pip \
    python3-dev \
    build-essential \
    git \
    curl \
    wget \
    vim \
    htop \
    net-tools

# 3. Upgrade pip
pip3 install --upgrade pip

# 4. Create directory structure
mkdir -p ~/logs
mkdir -p ~/backups
mkdir -p ~/scripts

# 5. Set timezone (optional)
sudo timedatectl set-timezone America/New_York

# 6. Configure automatic security updates (recommended)
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades

# 7. Check disk space
df -h

# 8. Check memory
free -h

# 9. Check CPU
lscpu
```

---

### Step 0.10: Configure AWS CLI (for S3 access)

**If you created an S3 bucket, configure AWS CLI:**

1. **Install AWS CLI**
   ```bash
   sudo apt install -y awscli
   ```

2. **Configure AWS Credentials**
   ```bash
   aws configure
   ```

3. **Enter Information**
   ```
   AWS Access Key ID: [Your access key]
   AWS Secret Access Key: [Your secret key]
   Default region name: us-west-2 (match your EC2 region)
   Default output format: json
   ```

4. **Get AWS Credentials:**
   - AWS Console → Top right (your name) → **"Security credentials"**
   - Scroll to **"Access keys"**
   - Click **"Create access key"**
   - Choose **"Command Line Interface (CLI)"**
   - Check: **"I understand..."**
   - Click **"Create access key"**
   - **Save both keys immediately!** (shown only once)

5. **Test S3 Access**
   ```bash
   # List buckets
   aws s3 ls
   
   # You should see: spyder-database-backups-yourname
   ```

---

### Step 0.11: Summary & Next Steps

✅ **You now have:**
- AWS account created and activated
- EC2 instance running (t3.2xlarge, Ubuntu 22.04)
- Static public IP (Elastic IP)
- Security groups configured (SSH + 4 service ports)
- S3 bucket for backups
- SSH access to your server
- AWS CLI configured

📋 **What you've spent:**
- EC2 instance: ~$0.33/hour = ~$240/month
- Elastic IP: Free (while associated)
- S3 storage: ~$0.023/GB/month (first 50 GB)
- Data transfer: Free tier covers most usage

🎯 **Next:**
- Continue to **Section 1** to configure your SPYDER services
- Or jump to **Section 3** to start installing applications

---

## 1. AWS EC2 Instance Configuration

### Recommended Instance Type
- **Instance Type:** `t3.2xlarge` or higher
- **vCPUs:** 8 cores minimum
- **RAM:** 32 GB minimum
- **Storage:** 100 GB EBS (gp3 recommended)

### Instance Configuration
```yaml
AMI: Ubuntu 22.04 LTS (HVM), SSD Volume Type
Architecture: x86_64 (64-bit)
Kernel: 6.8.0-1040-aws or later
Region: Any (us-west-2 recommended for low latency)
Availability Zone: Single-AZ deployment acceptable
```

### Storage Configuration
```
Root Volume: 
  - Size: 100 GB
  - Type: gp3
  - IOPS: 3000 (default)
  - Throughput: 125 MB/s
  - Encryption: Enabled (recommended)
```

---

## 2. Security Groups & Networking

### Inbound Rules (Security Group Configuration)

| Protocol | Port Range | Source | Description |
|----------|-----------|---------|-------------|
| SSH | 22 | Your IP/0.0.0.0/0 | Remote access |
| HTTP | 80 | 0.0.0.0/0 | Web interface (if using proxy) |
| Custom TCP | 8046 | 0.0.0.0/0 | SPYDER Main (curator1) |
| Custom TCP | 8055 | 0.0.0.0/0 | Codex Standalone |
| Custom TCP | 8056 | 0.0.0.0/0 | Codex V2 Orchestrator |
| Custom TCP | 8057 | 0.0.0.0/0 | Book Catalog |

### Outbound Rules
```
All traffic allowed (0.0.0.0/0) for:
- API calls (Anthropic, OpenAI)
- Package downloads (apt, pip)
- S3 access (if using remote storage)
```

### Elastic IP (Recommended)
```
Allocate and associate an Elastic IP for:
- Stable endpoint for services
- DNS mapping (optional)
- Consistent access during restarts
```

---

## 3. Operating System Setup

### Base System Configuration
```bash
Operating System: Ubuntu 22.04.5 LTS
Kernel: 6.8.0-1040-aws
Python Version: 3.10.12
```

### Required System Packages
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install essential packages
sudo apt install -y \
    python3-pip \
    python3-dev \
    build-essential \
    git \
    curl \
    wget \
    vim \
    htop \
    net-tools \
    nginx (optional - for reverse proxy)
```

### Python Environment
```bash
# Upgrade pip
pip3 install --upgrade pip

# Core Python packages
pip3 install \
    anthropic==0.68.0 \
    lancedb==0.25.0 \
    flask==3.0.0 \
    pandas==2.0.3 \
    openai==1.108.0 \
    boto3 \
    pyarrow
```

---

## 4. Directory Structure

### Main Directory Layout
```
/home/ubuntu/
├── spyder_lancedb_development/          # Main database (shared by all services)
│   ├── books.lance/
│   ├── authors.lance/
│   ├── quality_evaluations_real.lance/
│   └── processing_history.lance/
│
├── agenticspyder4_curator1_aws/         # Port 8046 - Main SPYDER
│   ├── src/
│   │   ├── agenticspyder4_advanced_lancedb.py
│   │   ├── lancedb_search_engine.py
│   │   ├── intent_classifier.py
│   │   └── [54 Python modules]
│   ├── templates/
│   │   └── agenticspyder4_curator1.html (99KB)
│   └── requirements.txt
│
├── codex_standalone/                     # Port 8055 - Standalone AI
│   ├── codex_app.py
│   ├── extended_thinking.py
│   ├── intent_classifier.py
│   └── templates/
│
├── codex_v2_orchestrator/               # Port 8056 - Universal Orchestrator
│   ├── app.py
│   ├── orchestrator.py
│   └── templates/
│
├── spyder_main/                         # Port 8057 - Book Catalog
│   ├── spyder_book_catalog_enhanced_fixed.py
│   └── templates/
│
├── logs/                                # Centralized logging
│   ├── spyder_main.log
│   ├── codex.log
│   └── catalog.log
│
├── backups/                             # Database backups
│   └── spyder_lancedb_YYYYMMDD.tar.gz
│
└── scripts/                             # Management scripts
    ├── start_spyder_fixed.py
    └── health_check.sh
```

### Directory Permissions
```bash
Owner: ubuntu:ubuntu
Permissions: 755 for directories, 644 for files
Executables: 755 (startup scripts)
```

---

## 5. Database Configuration

### LanceDB Structure
```
Location: /home/ubuntu/spyder_lancedb_development/
Type: LanceDB (vector database)
Size: ~650 GB (uncompressed)
Format: Apache Arrow / Parquet
```

### Database Tables

#### books.lance
```yaml
Purpose: Main book collection
Records: 302 books
Fields:
  - id (string)
  - title (string)
  - author (string)
  - category (string)
  - themes (list)
  - embedding (vector[3072])
  - enrichment_data (json)
```

#### authors.lance
```yaml
Purpose: Author metadata
Records: 580 authors
Fields:
  - id (string)
  - name (string)
  - expertise_score (float 0-10)
  - institution (string)
  - research_areas (list)
```

#### quality_evaluations_real.lance
```yaml
Purpose: Book quality ratings
Records: 302 entries
Fields:
  - book_id (string)
  - quality_score (float 0-10)
  - review_source (string)
  - evaluation_date (timestamp)
```

### Database Access Pattern
```
Single shared database accessed by all services
Read-only for most services
Write access: controlled through specific APIs
Connection: Direct file system access (local)
```

---

## 6. Service Port Mapping

### Active Services

| Port | Service | Description | Process | Memory |
|------|---------|-------------|---------|--------|
| 8046 | SPYDER Main | curator1 - Main research interface | Python Flask | ~800 MB |
| 8055 | Codex Standalone | AI without database | Python Flask | ~300 MB |
| 8056 | Codex V2 | Universal orchestrator | Python Flask | ~400 MB |
| 8057 | Book Catalog | Browse/manage books | Python Flask | ~600 MB |

### Port Configuration Commands
```bash
# Check active ports
sudo netstat -tlnp | grep -E "8046|8055|8056|8057"

# Or with ss (modern alternative)
sudo ss -tlnp | grep -E "8046|8055|8056|8057"
```

### Firewall Rules (UFW - if used)
```bash
sudo ufw allow 8046/tcp
sudo ufw allow 8055/tcp
sudo ufw allow 8056/tcp
sudo ufw allow 8057/tcp
```

---

## 7. Systemd Service Management

### Service Files Location
```
/etc/systemd/system/
├── spyder-main.service        # Port 8046
├── spyder-catalog.service     # Port 8057
└── spyder-backtest.service    # Experimental
```

### Example Service File Template

**File:** `/etc/systemd/system/spyder-main.service`
```ini
[Unit]
Description=SPYDER Main Research Interface (Port 8046)
After=network.target

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/agenticspyder4_curator1_aws/src
Environment="PATH=/usr/local/bin:/usr/bin:/bin"
Environment="ANTHROPIC_API_KEY=YOUR_KEY_HERE"
Environment="PYTHONUNBUFFERED=1"
ExecStart=/usr/bin/python3 agenticspyder4_advanced_lancedb.py
Restart=on-failure
RestartSec=10s
StandardOutput=append:/home/ubuntu/logs/spyder_main.log
StandardError=append:/home/ubuntu/logs/spyder_main_error.log

[Install]
WantedBy=multi-user.target
```

### Service Management Commands
```bash
# Enable service to start on boot
sudo systemctl enable spyder-main.service

# Start service
sudo systemctl start spyder-main.service

# Check status
sudo systemctl status spyder-main.service

# View logs
journalctl -u spyder-main.service -f

# Restart service
sudo systemctl restart spyder-main.service

# Stop service
sudo systemctl stop spyder-main.service
```

---

## 8. Environment Variables & API Keys

### API Key Storage Methods

#### Method 1: File-based (Recommended)
```bash
# Create API key file
vi ~/.anthropic_key

# Content:
export ANTHROPIC_API_KEY="sk-ant-api03-YOUR_KEY_HERE"
export OPENAI_API_KEY="sk-YOUR_OPENAI_KEY_HERE"

# Set permissions
chmod 600 ~/.anthropic_key

# Source in .bashrc
echo "source ~/.anthropic_key" >> ~/.bashrc
```

#### Method 2: Environment Variables
```bash
# Add to ~/.bashrc or ~/.profile
export ANTHROPIC_API_KEY="your_key_here"
export OPENAI_API_KEY="your_key_here"
```

#### Method 3: Systemd Service Environment
```ini
# In service file [Service] section
Environment="ANTHROPIC_API_KEY=your_key_here"
Environment="OPENAI_API_KEY=your_key_here"
```

### AWS Credentials (for S3 access)
```bash
# Configure AWS CLI
aws configure

# Credentials stored in:
~/.aws/credentials

# Format:
[default]
aws_access_key_id = YOUR_ACCESS_KEY
aws_secret_access_key = YOUR_SECRET_KEY
region = us-west-2
```

---

## 9. Process Management

### Manual Process Management

#### Starting Services Manually
```bash
# Port 8046 - SPYDER Main
cd /home/ubuntu/agenticspyder4_curator1_aws/src
nohup python3 agenticspyder4_advanced_lancedb.py > /home/ubuntu/logs/spyder.log 2>&1 &

# Port 8055 - Codex Standalone
cd /home/ubuntu/codex_standalone
nohup python3 codex_app.py > /home/ubuntu/logs/codex.log 2>&1 &

# Port 8056 - Codex V2
cd /home/ubuntu/codex_v2_orchestrator
nohup python3 app.py > /home/ubuntu/logs/codex_v2.log 2>&1 &

# Port 8057 - Book Catalog
cd /home/ubuntu/spyder_main
nohup python3 spyder_book_catalog_enhanced_fixed.py > /home/ubuntu/logs/catalog.log 2>&1 &
```

#### Checking Running Processes
```bash
# List SPYDER processes
ps aux | grep -E "spyder|curator|codex" | grep -v grep

# Get process IDs
pgrep -f "agenticspyder4"

# Check resource usage
htop
# OR
top -u ubuntu
```

#### Stopping Services
```bash
# Kill by process name
pkill -f "agenticspyder4_advanced_lancedb"
pkill -f "codex_app"

# Kill by PID
kill <PID>

# Force kill if needed
kill -9 <PID>
```

### Process Monitoring Script
```bash
# Create health check script
vi /home/ubuntu/scripts/health_check.sh

#!/bin/bash
# Check if all services are running
PORTS=(8046 8055 8056 8057)

for port in "${PORTS[@]}"; do
    if netstat -tuln | grep -q ":$port "; then
        echo "✅ Port $port: Running"
    else
        echo "❌ Port $port: Down"
        # Auto-restart logic here
    fi
done
```

---

## 10. Backup Strategy

### Database Backup

#### Manual Backup
```bash
# Full database backup
cd /home/ubuntu
tar -czf backups/spyder_lancedb_$(date +%Y%m%d_%H%M%S).tar.gz \
    spyder_lancedb_development/

# Size: ~650-700 GB compressed
# Time: 10-15 minutes
```

#### Automated Backup (Cron)
```bash
# Add to crontab
crontab -e

# Daily backup at 2 AM
0 2 * * * /home/ubuntu/scripts/backup_database.sh

# Backup script content:
#!/bin/bash
DATE=$(date +%Y%m%d)
tar -czf /home/ubuntu/backups/spyder_lancedb_$DATE.tar.gz \
    /home/ubuntu/spyder_lancedb_development/

# Keep last 7 days
find /home/ubuntu/backups -name "spyder_lancedb_*.tar.gz" -mtime +7 -delete
```

#### S3 Backup (Optional)
```bash
# Sync to S3
aws s3 sync /home/ubuntu/spyder_lancedb_development/ \
    s3://your-bucket/spyder-backup/ \
    --storage-class STANDARD_IA

# Restore from S3
aws s3 sync s3://your-bucket/spyder-backup/ \
    /home/ubuntu/spyder_lancedb_development/
```

### Configuration Backup
```bash
# Backup scripts and configs
tar -czf config_backup_$(date +%Y%m%d).tar.gz \
    ~/agenticspyder4_curator1_aws/src/*.py \
    ~/codex_standalone/ \
    ~/codex_v2_orchestrator/ \
    ~/spyder_main/ \
    ~/.anthropic_key \
    /etc/systemd/system/spyder*.service
```

---

## 11. Monitoring & Health Checks

### System Resource Monitoring

#### CPU & Memory
```bash
# Real-time monitoring
htop

# Check memory usage
free -h

# Check disk space
df -h

# IO statistics
iostat -x 1
```

#### Service Health Check
```bash
# Check all ports
netstat -tlnp | grep -E "8046|8055|8056|8057"

# HTTP health check
curl -I http://localhost:8046/curator1
curl -I http://localhost:8055
curl -I http://localhost:8056
curl -I http://localhost:8057
```

### Log Monitoring
```bash
# Follow logs in real-time
tail -f /home/ubuntu/logs/spyder.log
tail -f /home/ubuntu/logs/codex.log
tail -f /home/ubuntu/logs/catalog.log

# Check for errors
grep -i "error" /home/ubuntu/logs/*.log
grep -i "exception" /home/ubuntu/logs/*.log
```

### Performance Metrics

#### Database Size Monitoring
```bash
# Check database size
du -sh /home/ubuntu/spyder_lancedb_development/

# Check individual tables
du -sh /home/ubuntu/spyder_lancedb_development/*.lance/
```

#### Response Time Monitoring
```bash
# Test response time
time curl http://localhost:8046/curator1

# Load testing (optional)
# Install: sudo apt install apache2-utils
ab -n 100 -c 10 http://localhost:8046/curator1
```

### Automated Monitoring (Optional)

#### Create monitoring script
```bash
vi /home/ubuntu/scripts/monitor.sh

#!/bin/bash
# Monitor SPYDER services

LOG_FILE="/home/ubuntu/logs/monitor.log"

check_service() {
    PORT=$1
    NAME=$2
    
    if curl -s -o /dev/null -w "%{http_code}" http://localhost:$PORT | grep -q "200"; then
        echo "[$(date)] ✅ $NAME (port $PORT) - OK" >> $LOG_FILE
    else
        echo "[$(date)] ❌ $NAME (port $PORT) - DOWN" >> $LOG_FILE
        # Send alert or restart service
    fi
}

check_service 8046 "SPYDER Main"
check_service 8055 "Codex Standalone"
check_service 8056 "Codex V2"
check_service 8057 "Book Catalog"
```

#### Add to cron (every 5 minutes)
```bash
crontab -e

*/5 * * * * /home/ubuntu/scripts/monitor.sh
```

---

## Quick Reference Card

### Essential Commands
```bash
# Start all services
for port in 8046 8055 8056 8057; do 
    sudo systemctl start spyder-${port}.service
done

# Stop all services
for port in 8046 8055 8056 8057; do 
    sudo systemctl stop spyder-${port}.service
done

# Check status
ps aux | grep python3 | grep -E "8046|8055|8056|8057"

# View logs
tail -f /home/ubuntu/logs/*.log

# Backup database
tar -czf ~/backups/db_backup_$(date +%Y%m%d).tar.gz ~/spyder_lancedb_development/

# Check disk space
df -h /

# Check memory
free -h
```

### Troubleshooting

#### Service won't start
```bash
# 1. Check if port is already in use
sudo netstat -tlnp | grep <PORT>

# 2. Kill existing process
sudo kill -9 <PID>

# 3. Check logs for errors
tail -50 /home/ubuntu/logs/<service>.log

# 4. Verify API keys
echo $ANTHROPIC_API_KEY

# 5. Check file permissions
ls -la /home/ubuntu/<service_directory>
```

#### Out of disk space
```bash
# 1. Check disk usage
df -h

# 2. Find large files
du -sh /home/ubuntu/* | sort -rh | head -10

# 3. Clean old logs
find /home/ubuntu/logs -name "*.log" -mtime +30 -delete

# 4. Remove old backups
find /home/ubuntu/backups -name "*.tar.gz" -mtime +7 -delete
```

#### High memory usage
```bash
# 1. Check memory by process
ps aux --sort=-%mem | head -10

# 2. Restart services to free memory
sudo systemctl restart spyder-main.service

# 3. Check for memory leaks in logs
grep -i "memory" /home/ubuntu/logs/*.log
```

---

## Cost Estimation

### AWS EC2 Costs (us-west-2)
```
t3.2xlarge: ~$0.33/hour = ~$240/month (on-demand)
Reserved (1-year): ~$150/month (save 38%)
Reserved (3-year): ~$100/month (save 58%)

100 GB gp3 EBS: ~$8/month
Elastic IP: Free (when associated)
Data Transfer: ~$0.09/GB (first 10 TB)
```

### API Costs (Estimated Monthly)
```
Anthropic Claude:
  - Sonnet: ~$50-100/month (heavy usage)
  - Haiku: ~$10-20/month (standard usage)

OpenAI (if used):
  - Embeddings: ~$20-40/month
  - GPT-4: ~$100-200/month (if enabled)
```

### Total Estimated Cost
```
Infrastructure: $240/month (on-demand) or $100/month (reserved)
APIs: $60-120/month
Total: $160-360/month
```

---

## Security Best Practices

1. **API Keys:**
   - Store in separate files with 600 permissions
   - Never commit to git
   - Rotate keys quarterly

2. **SSH Access:**
   - Use SSH keys (disable password auth)
   - Restrict to specific IP ranges
   - Use bastion host for production

3. **Security Groups:**
   - Limit inbound to required ports only
   - Use source IP restrictions when possible
   - Enable CloudWatch logs

4. **System Updates:**
   - Regular `apt update && apt upgrade`
   - Monitor security patches
   - Test updates in dev first

5. **Backups:**
   - Automated daily backups
   - Off-site storage (S3)
   - Test restore procedures monthly

---

## Support & Troubleshooting

### Common Issues

**Issue:** Services crash on startup
- **Cause:** Missing API keys or dependencies
- **Solution:** Check logs, verify environment variables

**Issue:** Slow response times
- **Cause:** High memory usage or disk I/O
- **Solution:** Restart services, check resource usage

**Issue:** Database corruption
- **Cause:** Unclean shutdown or disk errors
- **Solution:** Restore from latest backup

### Log Locations
```
System logs: /var/log/syslog
Service logs: /home/ubuntu/logs/*.log
Systemd logs: journalctl -u spyder-*.service
```

### Getting Help
- Check logs first: `/home/ubuntu/logs/`
- Review systemd status: `systemctl status spyder-*.service`
- Monitor resources: `htop`, `df -h`, `free -h`

---

**Document Version:** 1.0  
**Last Updated:** October 2025  
**Platform:** AWS EC2 + Ubuntu 22.04 LTS  
**Services:** SPYDER Research Platform (4 services)

