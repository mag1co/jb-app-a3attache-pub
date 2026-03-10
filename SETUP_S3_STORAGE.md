# S3-Compatible Storage Setup Guide for A³ Attache

Complete step-by-step guide to configure Amazon S3, MinIO, or other S3-compatible storage for use with A³ Attache YouTrack app.

---

## Table of Contents

### Getting Started
1. [Overview](#overview)
2. [Choose Your Storage Provider](#choose-your-storage-provider)

### Part A: Amazon S3 Setup
3. [AWS Prerequisites](#aws-prerequisites)
4. [AWS Step 1: Create S3 Bucket](#aws-step-1-create-s3-bucket)
5. [AWS Step 2: Create IAM User](#aws-step-2-create-iam-user)
6. [AWS Step 3: Generate Access Keys](#aws-step-3-generate-access-keys)
7. [AWS Step 4: Configure CORS](#aws-step-4-configure-cors-optional)

### Part B: MinIO Setup
8. [MinIO Prerequisites](#minio-prerequisites)
9. [MinIO Step 1: Install MinIO Server](#minio-step-1-install-minio-server-optional)
10. [MinIO Step 2: Create Bucket](#minio-step-2-create-bucket)
11. [MinIO Step 3: Create Access Keys](#minio-step-3-create-access-keys)
12. [MinIO Step 4: Configure CORS](#minio-step-4-configure-cors-optional)

### Common Configuration
13. [Configure A³ Attache](#configure-a3-attache)
14. [Test Configuration](#test-configuration)

### Additional Resources
15. [Troubleshooting](#troubleshooting)
16. [How to Find Files After Issue Deletion](#how-to-find-files-after-issue-deletion)
17. [Security Best Practices](#security-best-practices)
18. [Cost Estimation](#cost-estimation)

---

## Overview

**A³ Attache** supports any S3-compatible storage service, including:

- **Amazon S3** — Cloud storage service from AWS (pay-as-you-go)
- **MinIO** — Self-hosted, open-source S3-compatible storage (free, on-premises)
- **Wasabi** — Cloud storage with predictable pricing
- **DigitalOcean Spaces** — Cloud storage from DigitalOcean
- **Backblaze B2** — Low-cost cloud storage
- **Other S3 API-compatible services**

This guide covers:
- **Amazon S3** — Full setup from scratch
- **MinIO** — Self-hosted deployment and configuration

---

## Choose Your Storage Provider

### When to choose AWS S3
- Need a fully managed service with high availability and SLA
- Global availability and easy CDN integration are important
- Require enterprise-grade security and compliance
- Prefer not to manage infrastructure

### When to choose MinIO
- On‑prem/private cloud with full control over data and costs
- No external internet or strict data residency requirements
- Need high performance and flexible tuning
- Require S3 API compatibility without vendor lock‑in

### Useful links
- AWS S3 pricing: https://aws.amazon.com/s3/pricing/
- MinIO documentation: https://min.io/docs/

**→ Follow [Part A: Amazon S3 Setup](#part-a-amazon-s3-setup)**

**→ Follow [Part B: MinIO Setup](#part-b-minio-setup)**

---

# Part A: Amazon S3 Setup

## AWS Prerequisites

- Active AWS account with billing enabled
- Access to AWS Management Console
- YouTrack Server installation with A³ Attache app installed
- Administrator permissions in YouTrack

---

## AWS Step 1: Create S3 Bucket

### 1.1. Navigate to S3 Console

1. Log in to [AWS Management Console](https://console.aws.amazon.com/)
2. Search for **"S3"** in the services search bar
3. Click **"Create bucket"**

### 1.2. Configure Bucket Settings

**General configuration:**
- **Bucket name**: Choose a unique name (e.g., `youtrack-attachments-prod`)
  - Must be globally unique across all AWS accounts
  - Use lowercase letters, numbers, and hyphens only
  - Example: `a3attache-demo-bucket`
- **AWS Region**: Select the region closest to your YouTrack server
  - Examples: `us-east-1`, `eu-west-1`, `eu-north-1`, `ap-southeast-1`
  - **Remember this region** — you'll need it for A³ Attache configuration

**Object Ownership:**
- Select **"ACLs disabled (recommended)"**

**Block Public Access settings:**
- Keep **"Block all public access"** enabled (recommended)
- A³ Attache uses presigned URLs, so public access is NOT required

**Bucket Versioning:**
- Enable if you want to preserve file history (optional)
- Recommended: **Disabled** for simplicity

**Default encryption:**
- Select **"Server-side encryption with Amazon S3 managed keys (SSE-S3)"**
- Or use **AWS KMS** if you need advanced encryption control

### 1.3. Create Bucket

Click **"Create bucket"** at the bottom of the page.

**Note the following values** (you'll need them later):
- ✅ **Bucket name**: `a3attache-demo-bucket`
- ✅ **AWS Region**: `eu-north-1` (or your selected region)

---

## AWS Step 2: Create IAM User

### 2.1. Navigate to IAM Console

1. In AWS Console, search for **"IAM"**
2. In the left sidebar, click **"Users"**
3. Click **"Create user"**

### 2.2. Set User Details

**User name:** `a3attache-s3-user` (or any descriptive name)

**Provide user access to the AWS Management Console:**
- ❌ **Do NOT check this box** — this user is for API access only

Click **"Next"**

### 2.3. Set Permissions

**Option A: Full S3 Access (Simple)**

1. Select **"Attach policies directly"**
2. In the search box, type `S3`
3. Check **"AmazonS3FullAccess"**
4. Click **"Next"** → **"Create user"**

**Option B: Bucket-Specific Access (Recommended for Production)**

1. Select **"Attach policies directly"**
2. Click **"Create policy"** (opens in new tab)
3. Switch to **JSON** tab and paste:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "A3AttacheS3Access",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::a3attache-demo-bucket",
                "arn:aws:s3:::a3attache-demo-bucket/*"
            ]
        }
    ]
}
```

**Important:** Replace `a3attache-demo-bucket` with your actual bucket name.

4. Click **"Next: Tags"** → **"Next: Review"**
5. **Policy name**: `A3Attache-S3-Policy`
6. Click **"Create policy"**
7. Return to the user creation tab and refresh the policy list
8. Search for `A3Attache-S3-Policy` and select it
9. Click **"Next"** → **"Create user"**

---

## AWS Step 3: Generate Access Keys

### 3.1. Create Access Key

1. After user creation, click on the username (`a3attache-s3-user`)
2. Go to **"Security credentials"** tab
3. Scroll down to **"Access keys"** section
4. Click **"Create access key"**

### 3.2. Select Use Case

Select **"Application running outside AWS"**
- This is the correct option for YouTrack integration

Click **"Next"**

### 3.3. Set Description (Optional)

**Description tag**: `YouTrack A3 Attache Integration`

Click **"Create access key"**

### 3.4. Save Credentials

⚠️ **IMPORTANT:** Save these credentials immediately — the Secret Access Key will NOT be shown again!

Copy and save in a secure location:
- **Access key ID**: `AKIAIOSFODNN7EXAMPLE` (example)
- **Secret access key**: `wJXUtnFEMI/KENG/bPxRfiCYPLEKEY` (example)

Click **"Done"**

---

## AWS Step 4: Configure CORS (Optional)

A³ Attache uploads and downloads files directly from the browser using presigned URLs. For this to work, you need to configure CORS on your S3 bucket.

**📖 See detailed CORS setup instructions:** [SETUP_CORS.md](./SETUP_CORS.md#aws-s3-настройка-cors)

**Quick summary:**
- Allow your YouTrack domain in `AllowedOrigins`
- Include methods: `GET`, `PUT`, `POST`, `DELETE`, `HEAD`
- Include `Date` in `ExposeHeaders` for clock skew detection

---

---

# Part B: MinIO Setup

## MinIO Prerequisites

**For self-hosted MinIO:**
- Linux/Windows/macOS server with 2+ GB RAM
- Docker (recommended) or direct binary installation
- Network access from YouTrack server to MinIO server

**OR use MinIO Cloud:**
- Free trial available at https://min.io/

**Common requirements:**
- YouTrack Server installation with A³ Attache app installed
- Administrator permissions in YouTrack

---

## MinIO Step 1: Install MinIO Server (Optional)

**Skip this step if you're using MinIO Cloud or already have MinIO running.**

### Option A: Docker Installation (Recommended)

**1. Pull MinIO Docker image:**
```bash
docker pull minio/minio:latest
```

**2. Create data directory:**
```bash
mkdir -p ~/minio/data
```

**3. Run MinIO server:**
```bash
docker run -d \
  --name minio-server \
  -p 9000:9000 \
  -p 9001:9001 \
  -v ~/minio/data:/data \
  -e "MINIO_ROOT_USER=minioadmin" \
  -e "MINIO_ROOT_PASSWORD=minioadmin123" \
  minio/minio server /data --console-address ":9001"
```

**4. Access MinIO Console:**
- Open browser: `http://localhost:9001`
- Login: `minioadmin` / `minioadmin123`

**Important:** Change default credentials in production!

### Option B: Binary Installation (Linux)

**1. Download MinIO binary:**
```bash
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv minio /usr/local/bin/
```

**2. Create data directory:**
```bash
mkdir -p ~/minio/data
```

**3. Start MinIO:**
```bash
export MINIO_ROOT_USER=minioadmin
export MINIO_ROOT_PASSWORD=minioadmin123
minio server ~/minio/data --console-address ":9001"
```

**4. Access MinIO Console:**
- Open browser: `http://localhost:9001`
- Login: `minioadmin` / `minioadmin123`

### Option C: Docker Compose (Production)

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  minio:
    image: minio/minio:latest
    container_name: minio
    restart: unless-stopped
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin123
    volumes:
      - minio_data:/data
    command: server /data --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

volumes:
  minio_data:
    driver: local
```

Start MinIO:
```bash
docker-compose up -d
```

---

## MinIO Step 2: Create Bucket

### 2.1. Access MinIO Console

**Self-hosted:**
- URL: `http://localhost:9001` (or your server IP)
- Login: Your root credentials

**MinIO Cloud:**
- URL: https://cloud.min.io/
- Login: Your MinIO account

### 2.2. Create Bucket

1. Click **"Buckets"** in the left sidebar
2. Click **"Create Bucket"** button (top right)
3. Enter bucket details:
   - **Bucket Name**: `youtrack-attachments` (or any name)
     - Use lowercase letters, numbers, hyphens only
     - No spaces or special characters
   - **Versioning**: Disabled (recommended for simplicity)
   - **Object Locking**: Disabled
4. Click **"Create Bucket"**

**Note the following values:**
- ✅ **Bucket name**: `youtrack-attachments`
- ✅ **MinIO endpoint**: `http://localhost:9000` (or `https://your-server.com:9000`)

---

## MinIO Step 3: Create Access Keys

### 3.1. Navigate to Access Keys

1. In MinIO Console, click **"Access Keys"** in the left sidebar
2. Click **"Create access key"** button

### 3.2. Configure Access Key

**Option A: Full Access (Simple)**

1. Leave policy empty (inherits root permissions)
2. Click **"Create"**

**Option B: Bucket-Specific Access (Recommended)**

1. Click **"Restrict with policy"**
2. Paste this JSON policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::youtrack-attachments",
                "arn:aws:s3:::youtrack-attachments/*"
            ]
        }
    ]
}
```

**Important:** Replace `youtrack-attachments` with your actual bucket name.

3. Click **"Create"**

### 3.3. Save Credentials

⚠️ **IMPORTANT:** Save these credentials immediately — they will NOT be shown again!

**Copy and save securely:**
- **Access Key**: `minioadmin` (example: `Q3AM3UQ867SPQQA43P2F`)
- **Secret Key**: `minioadmin123` (example: `zuf+tfteSlswRu7BJ86wekitnifILbZam1KYY3TG`)

Click **"Close"** after saving.

---

## MinIO Step 4: Configure CORS (Optional)

A³ Attache uploads and downloads files directly from the browser using presigned URLs. For this to work, you need to configure CORS on your MinIO bucket.

**📖 See detailed CORS setup instructions:** [SETUP_CORS.md](./SETUP_CORS.md#minio-настройка-cors)

**Quick summary:**
- Allow your YouTrack domain in `AllowedOrigins`
- Include methods: `GET`, `PUT`, `POST`, `DELETE`, `HEAD`
- Include `Date` in `ExposeHeaders` for clock skew detection
- Use MinIO Console or `mc` client to apply configuration

---

# Common Configuration

## Configure A³ Attache

### 5.1. Open A³ Attache Settings in YouTrack

1. Log in to YouTrack as Administrator
2. Go to **Administration** → **Applications**
3. Find **"A³ Attache"** in the list
4. Click on it and go to **"Settings"** tab

### 5.2. Enter S3 Connection Settings

Configuration differs depending on your storage provider:

#### For Amazon S3

**■ S3 SERVER CONNECTION SETTINGS**

| Setting | Value | Example |
|---------|-------|---------|
| **S3 Endpoint** | `https://s3.<region>.amazonaws.com` | `https://s3.eu-north-1.amazonaws.com` |
| **Region** | Your AWS region | `eu-north-1` |
| **Bucket Name** | Your bucket name | `a3attache-demo-bucket` |
| **Base Directory** | Optional prefix | `youtrack-attachments` or leave empty |
| **Access Key ID** | From AWS Step 3.4 | `AKIAIOSFODNN7EXAMPLE` |
| **Secret Access Key** | From AWS Step 3.4 | `wJalrXUtnFEMI/K7MDENG/bPxRfiCY...` |

**■ ADVANCED S3 SETTINGS**

| Setting | Value for AWS | Notes |
|---------|---------------|---------|
| **Force Path Style** | `false` | AWS supports virtual-hosted style |
| **S3 Provider** | `aws_default` | Use `aws_strict` for strict compatibility |
| **Clock Skew Adjustment** | `0` | **Auto-detected** from S3 Date header; leave at 0 |
| **Disable Presign Caching** | `true` | Recommended |
| **Operating Mode** | `minimal` | Use `debug` for troubleshooting |

#### For MinIO

**■ S3 SERVER CONNECTION SETTINGS**

| Setting | Value | Example |
|---------|-------|---------|
| **S3 Endpoint** | `http://<server>:9000` or `https://...` | `http://localhost:9000` or `https://minio.example.com` |
| **Region** | Any value (MinIO ignores this) | `us-east-1` (recommended default) |
| **Bucket Name** | Your bucket name | `youtrack-attachments` |
| **Base Directory** | Optional prefix | `youtrack` or leave empty |
| **Access Key ID** | From MinIO Step 3.3 | `Q3AM3UQ867SPQQA43P2F` |
| **Secret Access Key** | From MinIO Step 3.3 | `zuf+tfteSlswRu7BJ86wekitnifILbZam1KYY3TG` |

**■ ADVANCED S3 SETTINGS**

| Setting | Value for MinIO | Notes |
|---------|----------------|---------|
| **Force Path Style** | `true` | **Required** for MinIO |
| **S3 Provider** | `minio_default` | Use `minio_strict` for strict mode |
| **Clock Skew Adjustment** | `0` | **Auto-detected** from S3 Date header; leave at 0 |
| **Disable Presign Caching** | `true` | Recommended |
| **Operating Mode** | `minimal` | Use `debug` for troubleshooting |

**Important notes for MinIO:**
- ⚠️ **Force Path Style must be `true`** — MinIO requires path-style URLs
- Use `http://` for local development, `https://` for production with SSL
- Include port number if not standard (80/443)
- Region can be any value, but `us-east-1` is recommended

Leave other settings at default values.

---

## Test Configuration

1. Scroll to the bottom of settings page
2. Click **"Run S3 Healthcheck"** button
3. Wait for the results

**Expected result:**
```
✅ S3 configuration is valid
✅ Successfully connected to bucket
✅ Write test: OK
✅ Read test: OK
✅ Delete test: OK
```

If you see errors, check [Troubleshooting](#troubleshooting) section below.

### 5.4. Save Settings

Click **"Save"** button at the top of the settings page.

---

## Troubleshooting

### Error: "Access Denied" or "403 Forbidden"

**Possible causes:**
1. IAM user doesn't have sufficient permissions
2. Bucket name is incorrect
3. Region mismatch

**Solution:**
- Verify IAM policy includes all required actions (`s3:PutObject`, `s3:GetObject`, `s3:DeleteObject`, `s3:ListBucket`)
- Check bucket name spelling (case-sensitive)
- Ensure the region in settings matches the bucket region

### Error: "SignatureDoesNotMatch"

**Possible causes:**
1. Incorrect Secret Access Key
2. Time skew between YouTrack server and S3
3. CORS not configured properly (missing `Date` header)

**Solution:**
- Re-check Secret Access Key (copy-paste carefully)
- Verify YouTrack server time is synchronized with NTP
- **Ensure CORS includes `Date` in `ExposeHeaders`** (see [SETUP_CORS.md](./SETUP_CORS.md))
- Clock skew is **auto-detected** from S3 responses; leave **Clock Skew Adjustment** at `0`
- Only manually adjust clock skew if auto-detection fails

### Error: "InvalidAccessKeyId"

**Possible causes:**
1. Incorrect Access Key ID
2. IAM user was deleted or disabled

**Solution:**
- Re-check Access Key ID
- Verify the IAM user exists and is active in AWS Console

### Error: "NoSuchBucket"

**Possible causes:**
1. Bucket name is incorrect
2. Bucket was deleted
3. Region mismatch

**Solution:**
- Verify bucket name in AWS S3 Console
- Ensure bucket exists in the specified region
- Check region setting in A³ Attache configuration

### Error: CORS-related issues in browser console

**Possible causes:**
1. CORS is not configured on the bucket
2. CORS configuration doesn't allow your YouTrack domain

**Solution:**
- Follow complete CORS setup guide: [SETUP_CORS.md](./SETUP_CORS.md)
- Add your specific YouTrack URL to `AllowedOrigins`

### Files upload slowly or timeout

**Possible causes:**
1. Network latency to storage server
2. Large file size
3. Bandwidth limitations

**Solution:**
- **AWS**: Choose region closer to your YouTrack server
- **MinIO**: Ensure MinIO server is on fast network connection
- Increase **Upload URL TTL** in A³ Attache settings (e.g., 1800 seconds = 30 minutes)
- Check network bandwidth and firewall rules

### Error: "Connection refused" or "Connection timeout" (MinIO)

**Possible causes:**
1. MinIO server is not running
2. Firewall blocking port 9000
3. Incorrect endpoint URL
4. Network connectivity issues

**Solution:**
- Check MinIO server status: `docker ps` or `systemctl status minio`
- Verify port 9000 (API) and 9001 (Console) are accessible
- Test connectivity: `curl http://localhost:9000/minio/health/live`
- Check firewall rules: `sudo ufw status` or `iptables -L`
- For Docker: ensure port mapping is correct (`-p 9000:9000`)

### Error: "Force Path Style required" (MinIO)

**Possible causes:**
1. **Force Path Style** setting is set to `false`
2. Using virtual-hosted style URL format

**Solution:**
- **Set Force Path Style to `true`** in A³ Attache settings
- This is **required** for MinIO compatibility
- MinIO does not support virtual-hosted style URLs

### Error: "X-Amz-Content-Sha256 mismatch" (MinIO)

**Possible causes:**
1. S3 Provider setting is incorrect
2. MinIO version compatibility issue

**Solution:**
- Set **S3 Provider** to `minio_default` or `minio_strict`
- Do NOT use `aws_default` for MinIO
- Update MinIO to latest version if possible

---

## AWS S3 Endpoint Reference by Region

| Region Name | Region Code | S3 Endpoint |
|-------------|-------------|-------------|
| US East (N. Virginia) | `us-east-1` | `https://s3.us-east-1.amazonaws.com` |
| US East (Ohio) | `us-east-2` | `https://s3.us-east-2.amazonaws.com` |
| US West (N. California) | `us-west-1` | `https://s3.us-west-1.amazonaws.com` |
| US West (Oregon) | `us-west-2` | `https://s3.us-west-2.amazonaws.com` |
| Europe (Ireland) | `eu-west-1` | `https://s3.eu-west-1.amazonaws.com` |
| Europe (London) | `eu-west-2` | `https://s3.eu-west-2.amazonaws.com` |
| Europe (Paris) | `eu-west-3` | `https://s3.eu-west-3.amazonaws.com` |
| Europe (Frankfurt) | `eu-central-1` | `https://s3.eu-central-1.amazonaws.com` |
| Europe (Stockholm) | `eu-north-1` | `https://s3.eu-north-1.amazonaws.com` |
| Asia Pacific (Tokyo) | `ap-northeast-1` | `https://s3.ap-northeast-1.amazonaws.com` |
| Asia Pacific (Seoul) | `ap-northeast-2` | `https://s3.ap-northeast-2.amazonaws.com` |
| Asia Pacific (Singapore) | `ap-southeast-1` | `https://s3.ap-southeast-1.amazonaws.com` |
| Asia Pacific (Sydney) | `ap-southeast-2` | `https://s3.ap-southeast-2.amazonaws.com` |
| Asia Pacific (Mumbai) | `ap-south-1` | `https://s3.ap-south-1.amazonaws.com` |
| South America (São Paulo) | `sa-east-1` | `https://s3.sa-east-1.amazonaws.com` |

For a complete list of regions, see [AWS Regional Services List](https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/).

## MinIO Endpoint Examples

| Deployment Type | Example Endpoint | Notes |
|-----------------|------------------|-------|
| Local Docker | `http://localhost:9000` | Default port 9000 |
| LAN Server | `http://192.168.1.100:9000` | Use internal IP |
| Public Server (HTTP) | `http://minio.example.com:9000` | Not recommended for production |
| Public Server (HTTPS) | `https://minio.example.com` | Recommended with SSL/TLS |
| Behind Reverse Proxy | `https://s3.example.com` | Nginx/Caddy with SSL |
| MinIO Cloud | `https://cloud.min.io` | Managed MinIO service |

---

## How to Find Files After Issue Deletion

When an issue is deleted in YouTrack, the associated files **remain in S3 storage**. A³ Attache organizes files using a specific folder structure that allows you to locate and recover them even after issue deletion.

### File Storage Structure

A³ Attache uses the following directory hierarchy:

```
<base-directory>/
  └── <storage-id>/
      └── <file-name>
```

**Example:**
- Base Directory: `youtrack-attachments`
- Storage ID: `a1b2c3d4-e5f6-7890-abcd-ef1234567890` (UUID assigned to issue)
- File path: `youtrack-attachments/a1b2c3d4-e5f6-7890-abcd-ef1234567890/document.pdf`

**Key points:**
- Each issue gets a unique **Storage ID** (UUID) when files are first attached
- The Storage ID is stored internally in issue's extension properties
- All files for an issue are stored in a folder named with this Storage ID
- Storage ID remains the same even if the issue is moved to another project
- Files are **NOT automatically deleted** when an issue is deleted in YouTrack

**Why UUID instead of Issue ID?**
- Ensures files remain accessible if issue is moved between projects
- Prevents conflicts if issue IDs are reused
- Provides stable storage location independent of issue metadata changes

### Metadata File (_meta.json)

Each storage folder contains a special `_meta.json` file with information about all files and the issue:

**Structure of _meta.json:**
```json
{
  "version": 1,
  "issueId": "TEST-123",
  "storageId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "idReadableHistory": [
    {
      "value": "TEST-123",
      "spaceKey": "TEST",
      "firstSeen": "2024-12-09T12:00:00.000Z",
      "lastSeen": "2024-12-09T14:30:00.000Z"
    }
  ],
  "files": [
    {
      "key": "document.pdf",
      "size": 1048576,
      "lastModified": "2024-12-09T12:00:00.000Z",
      "etag": "\"abc123...\"",
      "uploader": {
        "login": "john.doe",
        "fullName": "John Doe",
        "email": "john.doe@example.com",
        "uploadedAt": "2024-12-09T12:00:00.000Z"
      }
    }
  ],
  "fileCount": 1,
  "totalBytes": 1048576,
  "lastScanAt": "2024-12-09T14:30:00.000Z",
  "updatedAt": "2024-12-09T14:30:00.000Z"
}
```

**Key fields:**
- `issueId` — Current issue ID (e.g., `TEST-123`)
- `storageId` — UUID of the storage folder
- `idReadableHistory` — History of issue IDs (useful if issue was moved between projects)
- `files` — Array of all files with metadata and uploader information
- `uploader` — Information about who uploaded each file

**How to use _meta.json to find deleted issues:**

1. **Download _meta.json from a storage folder:**
   ```bash
   # AWS CLI
   aws s3 cp s3://your-bucket-name/youtrack-attachments/<storage-id>/_meta.json ./
   
   # MinIO Client
   mc cp myminio/youtrack-attachments/<storage-id>/_meta.json ./
   ```

2. **Read the file to find the issue ID:**
   ```bash
   cat _meta.json | grep "issueId"
   ```

3. **Or use jq for better formatting:**
   ```bash
   cat _meta.json | jq '.issueId, .idReadableHistory'
   ```

**Bulk search across all _meta.json files:**

```bash
# AWS CLI - find all _meta.json files and search for specific issue ID
aws s3 ls s3://your-bucket-name/youtrack-attachments/ --recursive | grep "_meta.json" | while read -r line; do
  key=$(echo $line | awk '{print $4}')
  aws s3 cp "s3://your-bucket-name/$key" - | grep -q "TEST-123" && echo "Found in: $key"
done

# MinIO Client - download all _meta.json files to search locally
mc find myminio/youtrack-attachments/ --name "_meta.json" --exec "mc cp {} ./meta-files/"
grep -r "TEST-123" ./meta-files/
```

---

### Method 1: Using AWS S3 Console

**Step-by-step:**

1. Open [AWS S3 Console](https://console.aws.amazon.com/s3/)
2. Click on your bucket name (e.g., `a3attache-demo-bucket`)
3. Navigate through folders:
   - Click on base directory (e.g., `youtrack-attachments/`)
   - You'll see folders with UUID names (e.g., `a1b2c3d4-e5f6-7890-abcd-ef1234567890/`)
   - Click on the UUID folder that corresponds to your issue
4. You'll see all files that were attached to that issue
5. To download:
   - Select file(s) → **Actions** → **Download**
   - Or click on file name → **Download** button

**How to find the Storage ID for a deleted issue:**
- The Storage ID is stored internally and not visible in YouTrack UI
- Use the `_meta.json` file in each storage folder to find the mapping between Storage ID and Issue ID
- See the Tips section below for search methods

**Bulk operations:**
- Select multiple files → **Actions** → **Download**
- To download entire issue folder: Use AWS CLI (see Method 2)

---

### Method 2: Using AWS CLI

**Prerequisites:**
- Install AWS CLI: https://aws.amazon.com/cli/
- Configure credentials: `aws configure`

**List all files for a specific issue (if you know the Storage ID):**
```bash
# Replace <storage-id> with the actual UUID from the A3 Storage ID field
aws s3 ls s3://your-bucket-name/youtrack-attachments/<storage-id>/
```

**Example:**
```bash
aws s3 ls s3://your-bucket-name/youtrack-attachments/a1b2c3d4-e5f6-7890-abcd-ef1234567890/
```

**Download all files from a deleted issue:**
```bash
# Download to local folder
aws s3 sync s3://your-bucket-name/youtrack-attachments/<storage-id>/ ./recovered-files/<storage-id>/
```

**Download a specific file:**
```bash
aws s3 cp s3://your-bucket-name/youtrack-attachments/<storage-id>/document.pdf ./
```

**List all storage folders (all issues with attachments):**
```bash
# See all UUID folders in base directory
aws s3 ls s3://your-bucket-name/youtrack-attachments/
```

**Search for files by name pattern across all issues:**
```bash
# Find all PDF files in all issues
aws s3 ls s3://your-bucket-name/youtrack-attachments/ --recursive | grep ".pdf"
```

---

### Method 3: Using MinIO Console

**Step-by-step:**

1. Open MinIO Console (e.g., `http://localhost:9001` or your server URL)
2. Login with your credentials
3. Click **"Buckets"** in the left sidebar
4. Click on your bucket name (e.g., `youtrack-attachments`)
5. Navigate through folders:
   - Click on base directory folder (if configured)
   - You'll see folders with UUID names (e.g., `a1b2c3d4-e5f6-7890-abcd-ef1234567890/`)
   - Click on the UUID folder that corresponds to your issue
6. You'll see all files attached to that issue
7. To download:
   - Click on file → **Download** button
   - Or select multiple files → **Download** button

**How to find the Storage ID:**
- The Storage ID is stored internally and not visible in YouTrack UI
- Use `_meta.json` files to find the mapping (see Tips section below)

**Bulk download:**
- Select multiple files using checkboxes
- Click **Download** button at the top

---

### Method 4: Using MinIO Client (mc)

**Prerequisites:**
- Install MinIO Client: https://min.io/docs/minio/linux/reference/minio-mc.html
- Configure alias (one-time setup):

```bash
mc alias set myminio http://localhost:9000 YOUR_ACCESS_KEY YOUR_SECRET_KEY
```

**List files for a specific issue (if you know the Storage ID):**
```bash
# Replace <storage-id> with the actual UUID
mc ls myminio/youtrack-attachments/<storage-id>/
```

**Example:**
```bash
mc ls myminio/youtrack-attachments/a1b2c3d4-e5f6-7890-abcd-ef1234567890/
```

**Download all files from deleted issue:**
```bash
# Mirror entire issue folder to local directory
mc mirror myminio/youtrack-attachments/<storage-id>/ ./recovered-files/<storage-id>/
```

**Download specific file:**
```bash
mc cp myminio/youtrack-attachments/<storage-id>/document.pdf ./
```

**List all storage folders (all issues with attachments):**
```bash
mc ls myminio/youtrack-attachments/
```

**Search for files recursively across all issues:**
```bash
# Find all PDF files in all issues
mc find myminio/youtrack-attachments/ --name "*.pdf"
```

---

### Method 5: Using S3 Browser Tools

**Third-party GUI tools for easier file management:**

#### For AWS S3:
- **S3 Browser** (Windows): https://s3browser.com/
- **Cyberduck** (Mac/Windows): https://cyberduck.io/
- **CloudBerry Explorer** (Windows): https://www.cloudberrylab.com/

#### For MinIO:
- **Cyberduck** (Mac/Windows): https://cyberduck.io/
- **MinIO Console** (built-in web UI)
- Any S3-compatible browser tool

**Configuration:**
- Enter your S3 endpoint, access key, and secret key
- Browse folders visually
- Drag-and-drop to download files

---

### Tips for Finding Files

#### If you don't know the Storage ID:

**1. Finding Storage ID for an issue:**
- The Storage ID is stored internally in YouTrack and not visible in the UI
- The best way to find it is through the `_meta.json` file (see method 2 below)

**2. Use _meta.json to find the Storage ID by Issue ID:**

Each storage folder contains a `_meta.json` file with the issue ID. You can search for a specific issue:

```bash
# AWS CLI - search for issue ID in all _meta.json files
aws s3 ls s3://your-bucket-name/youtrack-attachments/ | awk '{print $2}' | while read folder; do
  content=$(aws s3 cp "s3://your-bucket-name/youtrack-attachments/${folder}_meta.json" - 2>/dev/null)
  if echo "$content" | grep -q "TEST-123"; then
    echo "Issue TEST-123 found in folder: $folder"
    echo "$content" | jq '.storageId, .issueId, .files[].key'
  fi
done

# MinIO Client - download and search locally
mkdir -p ./temp-meta
mc find myminio/youtrack-attachments/ --name "_meta.json" --exec "mc cp {} ./temp-meta/"
grep -l "TEST-123" ./temp-meta/* | while read file; do
  echo "Found in: $file"
  cat "$file" | jq '.storageId, .issueId, .files'
done
```

**3. List all storage folders:**
```bash
# AWS CLI - shows all UUID folders
aws s3 ls s3://your-bucket-name/youtrack-attachments/

# MinIO Client
mc ls myminio/youtrack-attachments/
```

**4. Search by file modification date:**
```bash
# AWS CLI - files modified on specific date
aws s3 ls s3://your-bucket-name/youtrack-attachments/ --recursive | grep "2024-12-09"
```

**5. Search by file name across all issues:**
```bash
# AWS CLI
aws s3 ls s3://your-bucket-name/youtrack-attachments/ --recursive | grep "invoice"

# MinIO Client
mc find myminio/youtrack-attachments/ --name "*invoice*"
```

**6. Use S3 object metadata (if available):**
- Some S3 browsers show last modified date and file size
- Sort by date to find recently uploaded files

**7. Create a mapping of all issues (one-time):**

Download all `_meta.json` files and create a mapping:

```bash
# Create a CSV mapping of Storage ID → Issue ID
mkdir -p ./meta-backup
mc find myminio/youtrack-attachments/ --name "_meta.json" --exec "mc cp {} ./meta-backup/"

# Generate mapping
for file in ./meta-backup/*; do
  storageId=$(cat "$file" | jq -r '.storageId')
  issueId=$(cat "$file" | jq -r '.issueId')
  echo "$storageId,$issueId"
done > storage-mapping.csv

# Now you can search the CSV
grep "TEST-123" storage-mapping.csv
```

#### If you need to recover files from multiple deleted issues:

**Bulk download all files:**
```bash
# AWS CLI - download everything
aws s3 sync s3://your-bucket-name/youtrack-attachments/ ./recovered-files/

# MinIO Client
mc mirror myminio/youtrack-attachments/ ./recovered-files/
```

---

### Preventing Accidental Data Loss

**Best practices:**

1. **Enable Bucket Versioning** (recommended):
   - Protects against accidental file deletion
   - Allows recovery of previous file versions
   - See [Security Best Practices](#security-best-practices) section

2. **Set up automated backups**:
   - AWS: Use AWS Backup or S3 Replication
   - MinIO: Use `mc mirror` or MinIO replication

3. **Implement lifecycle policies**:
   - Archive old files to cheaper storage (S3 Glacier, MinIO tiering)
   - Auto-delete files after retention period (e.g., 365 days)

4. **Document your folder structure**:
   - Keep a record of project IDs and issue ID ranges
   - Makes recovery easier

5. **Use Base Directory**:
   - Set a base directory in A³ Attache settings (e.g., `youtrack-attachments`)
   - Keeps all YouTrack files organized in one place
   - Easier to backup and manage

---

### Important Notes

⚠️ **Files are NOT automatically deleted when:**
- An issue is deleted in YouTrack
- An issue is moved to another project
- The A³ Attache app is uninstalled

✅ **Files ARE automatically deleted when:**
- You manually delete them from S3/MinIO storage
- A lifecycle policy deletes them based on age
- You delete the entire bucket

� **About _meta.json:**
- Each storage folder contains a `_meta.json` file with issue metadata
- This file maps Storage ID to Issue ID and contains file information
- Use it to find which issue a storage folder belongs to
- The file is automatically updated when files are added/removed
- It tracks issue ID history if the issue was moved between projects

�💡 **Recommendations:**
- **Before deleting issues**: Download the `_meta.json` file from the issue's storage folder for future reference
- **For bulk recovery**: Download all `_meta.json` files to create a Storage ID → Issue ID mapping
- **Regular maintenance**: Review and clean up files from deleted issues to save storage costs
- **Lifecycle policies**: Set up auto-archive/delete rules for old files

---

## Security Best Practices

### For Both AWS S3 and MinIO

#### 1. Use Least-Privilege Access Policy

**AWS S3**: Create custom IAM policy (don't use `AmazonS3FullAccess`)  
**MinIO**: Create restricted access keys with bucket-specific policy

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::your-bucket-name",
                "arn:aws:s3:::your-bucket-name/*"
            ]
        }
    ]
}
```

#### 2. Enable Bucket Versioning

Protect against accidental deletions:
- **AWS**: Bucket **Properties** → **Bucket Versioning** → **Enable**
- **MinIO**: `mc version enable myminio/youtrack-attachments`

#### 3. Use Encryption at Rest

**AWS S3**:
- Bucket **Properties** → **Default encryption** → **Enable SSE-S3** (or SSE-KMS)

**MinIO**:
```bash
# Server-side encryption (requires MinIO with KMS)
mc encrypt set sse-s3 myminio/youtrack-attachments
```

#### 4. Restrict CORS Origins

Replace `"*"` with your specific YouTrack domain:

```json
"AllowedOrigins": [
    "https://youtrack.yourcompany.com"
]
```

#### 5. Rotate Access Keys Regularly

- Create new access keys every 90 days
- Delete old keys after updating A³ Attache settings
- **AWS**: IAM → Users → Security credentials → Create access key
- **MinIO**: Console → Access Keys → Create new key

#### 6. Use Base Directory

Isolate YouTrack files in a dedicated folder:
- Set **Base Directory**: `youtrack-attachments`
- Keeps bucket organized
- Easier lifecycle policies
- Simpler backup strategies

#### 7. Use HTTPS/TLS

**AWS S3**: Always use `https://` endpoints (enforced by default)

**MinIO**: Configure SSL/TLS certificate:
```bash
# Place cert and key in MinIO certs directory
mkdir -p ~/.minio/certs
cp server.crt ~/.minio/certs/public.crt
cp server.key ~/.minio/certs/private.key
```

Or use reverse proxy (Nginx, Caddy, Traefik) with Let's Encrypt.

### AWS-Specific Best Practices

#### 8. Enable CloudTrail Logging

Monitor S3 API calls:
- AWS CloudTrail → Create trail → Enable S3 data events
- Set up CloudWatch alarms for suspicious activity

#### 9. Enable S3 Block Public Access

Keep **Block all public access** enabled (default)
- A³ Attache uses presigned URLs, so public access is NOT needed

#### 10. Use VPC Endpoints (Optional)

For YouTrack on AWS EC2:
- Create VPC Endpoint for S3
- Reduces data transfer costs
- Improves security (traffic stays within AWS)

### MinIO-Specific Best Practices

#### 11. Change Default Credentials

**Never use default** `minioadmin/minioadmin123` in production!

```bash
# Set strong credentials
export MINIO_ROOT_USER=your-secure-username
export MINIO_ROOT_PASSWORD=your-strong-password-min-8-chars
minio server /data
```

#### 12. Enable MinIO Audit Logs

```bash
# Enable audit logging
export MINIO_AUDIT_WEBHOOK_ENABLE_target1=on
export MINIO_AUDIT_WEBHOOK_ENDPOINT_target1=http://your-log-server:9000
```

#### 13. Use MinIO Console Behind Authentication

For production:
- Disable public access to port 9001 (console)
- Use VPN or SSH tunnel for admin access
- Or configure reverse proxy with authentication (Authelia, OAuth2 Proxy)

#### 14. Set Up Regular Backups

```bash
# Mirror bucket to backup location
mc mirror --watch myminio/youtrack-attachments backup/youtrack-attachments
```

Or use MinIO's built-in replication:
```bash
# Set up remote replication
mc replicate add myminio/youtrack-attachments --remote-bucket backup-bucket
```

#### 15. Run MinIO Behind Reverse Proxy

Use Nginx, Caddy, or Traefik:
- Automatic SSL with Let's Encrypt
- Rate limiting
- IP whitelisting
- Better logging


---

## Next Steps

After successful configuration:

1. **Attach files to issues** — Open any issue and use the A³ Attache widget in issue footer
2. **Monitor usage**:
   - **AWS**: Check S3 Console → Metrics
   - **MinIO**: Check MinIO Console → Monitoring
3. **Set lifecycle policies** (optional):
   - Automatically archive or delete old files after X days
   - **AWS**: S3 Lifecycle rules
   - **MinIO**: ILM policies via `mc ilm`
4. **Configure backups**:
   - **AWS**: Enable Versioning or use AWS Backup
   - **MinIO**: Set up `mc mirror` or replication
5. **Review costs**:
   - **AWS**: Monitor AWS Cost Explorer monthly
   - **MinIO**: Monitor server resource usage

---

## Support and Resources

### For A³ Attache Issues
- Check [Troubleshooting](#troubleshooting) section above
- Enable **Debug** mode in app settings for detailed logs
- Check YouTrack server logs
- Contact app support: [apps@mag1.cc](mailto:apps@mag1.cc)

### For AWS S3 Issues
- [AWS S3 Documentation](https://docs.aws.amazon.com/s3/)
- [AWS S3 FAQ](https://aws.amazon.com/s3/faqs/)
- [AWS Support Center](https://console.aws.amazon.com/support/)
- [AWS re:Post Community](https://repost.aws/)

### For MinIO Issues
- [MinIO Documentation](https://min.io/docs/minio/linux/index.html)
- [MinIO GitHub Issues](https://github.com/minio/minio/issues)
- [MinIO Slack Community](https://min.io/slack)
- [MinIO Support](https://min.io/support)

### Additional Resources
- [A³ Attache Documentation](README.md)
- [YouTrack Apps Documentation](https://www.jetbrains.com/help/youtrack/devportal/)
- [S3 API Compatibility](https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html)

---

**Last updated:** October 24, 2025  
**Document version:** 2.1  
**Compatible with:** A³ Attache v1.2.1+  
**Covers:** Amazon S3, MinIO, and other S3-compatible storage services

**Changelog v2.1:**
- Added `Date` header to CORS `ExposeHeaders` for automatic clock skew detection
- Updated Clock Skew Adjustment description to reflect auto-detection feature
