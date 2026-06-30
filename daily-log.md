## Day 1 — 19-06-2026

**Status:** ✅ DSA | ✅ SQL | ✅ System Design | ✅ Cloud/Git 

### DSA — Two Pointers
- Solved: LC-1 Two Sum, LC-125 Valid Palindrome, LC-167 Two Sum II
- Key takeaway: Two pointers works when array is sorted and you're looking 
  for a pair/triplet — avoids O(n²) brute force, gets to O(n).
- Mistake I made: Initially tried nested loops on LC-167 before remembering 
  array was sorted — should trigger two pointers immediately.

### SQL — SELECT + WHERE + ORDER BY
- Solved: LC-175 Combine Two Tables
- Key takeaway: LEFT JOIN needed here (not INNER) since we want all persons 
  even without an address.

### System Design — Scale, Load, Latency Basics
- Topics covered: Vertical vs Horizontal scaling, Latency vs Throughput, SLA/SLO
- One-line summary: Horizontal scaling = more machines (better for fault 
  tolerance), vertical = bigger machine (simpler but has a ceiling).

### Cloud/Git — Setup
- Created `my-prep-journey` repo with README, pushed first commit
- Created `cloudvault-app` repo with `.gitignore` (Python + AWS secrets excluded)
- Configured Git basics, comfortable with add/commit/push loop

### Reflection
- Total study time today: ~X hrs
- Confidence level (1-10): 
- Tomorrow's focus: Day 2 — Sliding Window + DNS/CDN, AWS Free Tier account + IAM user setup

## Day 2 — 20-06-2026

**Status:** ✅ Cloud/Git

### Cloud/Git — AWS Account + IAM Setup
- Created AWS Free Tier account, set a $1 billing alert as a safety net
- Created IAM non-root user (`cloudvault-admin`) instead of using root for daily work — 
  first hands-on security principle of the project
- Attached `AdministratorAccess` for now (flagged to scope down to least-privilege once 
  exact services in use are finalized — e.g. just S3, Lambda, EC2, RDS, VPC)
- Enabled MFA on the IAM user (not just root)
- Generated Access Key + Secret Key, configured locally via `aws configure` — credentials 
  live in `~/.aws/credentials`, never inside the project folder, so they can't accidentally 
  get committed
- Verified setup with `aws sts get-caller-identity` — confirmed CLI is authenticating as 
  the IAM user, not root
- Created feature branch `feature/iam-setup`, committed IAM setup notes to README, 
  opened and merged first real Pull Request
- Synced local `main` after merge with `git pull origin main`

### Key takeaway
- IAM Billing/Cost Explorer access is a separate, account-level toggle (off by default) — 
  even an IAM user with `AdministratorAccess` can't see Billing/Cost data until **root** 
  explicitly enables "IAM User and Role Access to Billing Information"
- Some AWS Console homepage widgets (ServiceCatalog, Security Hub) show "Access Denied" 
  by default and are unrelated to my actual IAM permissions — safe to ignore since they're 
  not part of the CloudVault architecture

### Mistake I made / confusion clarified
- Ran `git checkout main` after merging a PR on GitHub's website, and it said 
  "Your branch is up to date" — which was misleading. Local Git only knows about 
  remote changes after an explicit `fetch`/`pull`; merging on GitHub doesn't auto-sync 
  the local copy. Fixed by running `git pull origin main`.

### Concept learned (self-study)
- **Git merge vs rebase:**
  - `merge` combines two branch histories with a new merge commit — preserves full, 
    real history (used this for my actual PR merges)
  - `rebase` replays my commits on top of the latest target branch — cleaner, linear 
    history, but rewrites commit hashes; never do this on commits already pushed/shared

### Reflection
- Tomorrow's focus: Day 3 — S3 bucket setup + boto3 listing script

## Day 3 — 22-06-2026

**Status:** ✅ Cloud/Git

### Cloud/Git — S3 Bucket Setup
- Created `cloudvault-files` S3 bucket with versioning enabled
- Used AWS's new **Account Regional Namespace** feature (launched March 2026) instead of 
  the classic global namespace — avoids bucket name collisions, bucket name only needs 
  to be unique within my own account + region
- Blocked all public access — files will only be reachable via presigned URLs, not direct links
- Wrote `list_bucket.py` using boto3 to list bucket contents — confirms IAM user creds 
  (via `aws configure`) are working correctly end-to-end
- Created feature branch `feature/s3-bucket-setup`, committed, pushed, opened and merged 
  first "real" PR (previous one was just docs)

### Key takeaway
- Account Regional Namespace buckets generate longer auto-suffixed names 
  (`cloudvault-files-<account-id>-ap-south-1-an`) — needed to copy the *full* generated 
  name into my scripts, not just the prefix I typed

### Reflection
- Tomorrow's focus: Day 4 — Flask upload/download routes + presigned URLs

## Day 4 — 23-06-2026

**Status:** ✅ Cloud/Git

### Cloud/Git — File Upload API
- Built Flask app (`app.py`) with two routes:
  - `POST /upload` — saves incoming file directly to S3 via `upload_fileobj`
  - `GET /download/<filename>` — generates a presigned URL (1 hour expiry) for secure, 
    time-limited access to a private S3 object
- Tagged stable milestone: `git tag v0.1`

### Mistake I made (good interview story)
- Initial presigned download URLs failed with `SignatureDoesNotMatch`
- Root cause: boto3 was signing requests using the **global S3 endpoint** 
  (`s3.amazonaws.com`) instead of the **regional endpoint** (`s3.ap-south-1.amazonaws.com`) 
  — a known boto3 quirk for any region other than us-east-1, made worse since I'm using 
  the newer Account Regional Namespace bucket type
- Fix: explicitly passed `endpoint_url='https://s3.ap-south-1.amazonaws.com'` when 
  creating the boto3 S3 client
- Also hit a smaller issue along the way: PowerShell's `curl` is just an alias for 
  `Invoke-WebRequest`, not real curl — had to call `curl.exe` explicitly to use 
  `-X` / `-F` flags for testing file upload

### Key takeaway
- Presigned URLs are signed against the *exact* request — manually editing a URL 
  (e.g. fixing a typo in the filename) invalidates the signature; always regenerate 
  instead of hand-editing

### Reflection
- Tomorrow's focus: Day 5 — Lambda auto-expiry for uploaded files (EventBridge cron)

## Day 5 — 24-06-2026

**Status:** ✅ Cloud/Git

### Cloud/Git — Lambda Auto-Expiry + EventBridge Scheduler

**Goal:** Auto-delete files from S3 after 24 hours, using a Lambda function triggered 
on a recurring schedule.

#### What I Built
1. Created `auto_expiry.py` — Lambda function that:
   - Lists all objects in the `cloudvault-files` bucket
   - Checks each file's `LastModified` timestamp against a 24-hour cutoff
   - Deletes anything older than `EXPIRY_HOURS = 24`
2. Created Lambda function `cloudvault-auto-expiry` in AWS Console (Python 3.14 runtime)
3. Attached an **inline IAM policy** to the Lambda's execution role, scoped to only 
   `s3:ListBucket` + `s3:DeleteObject` on my specific bucket — least privilege in practice, 
   not just theory
4. Set up the trigger using **EventBridge Scheduler** (not the older "EventBridge Rules" — 
   AWS now recommends Scheduler for new work) with cron expression `0 * * * ? *` 
   → fires once every hour, on the hour, in UTC
5. Verified actual execution via CloudWatch Logs, confirmed UTC → IST conversion 
   (19:30 UTC = 1:00 AM IST)

---

### Mistakes Made (3 real bugs, all fixed — good interview material)

**Bug 1 — `Runtime.ImportModuleError: No module named 'auto_expiry'`**
- Cause: Lambda's "Handler" setting didn't match my actual filename. Handler format is 
  always `<filename-without-.py>.<function-name>` — mine was still pointing at the 
  default `lambda_function.lambda_handler` while my code was in `auto_expiry.py`.
- Fix: Set Handler (Configuration → General configuration) to exactly `auto_expiry.lambda_handler`, 
  matching my actual filename + function name. **Critical extra step:** had to click 
  **Deploy** again afterward — editing the handler setting alone doesn't apply until you deploy.

**Bug 2 — `ParamValidationError: Invalid bucket name "...-an "` (trailing space)**
- Cause: Copy-pasted the bucket name from the AWS Console into my code, and a stray 
  trailing space came along with it. S3 bucket names can't contain spaces, so boto3 
  rejected it before even calling AWS.
- Fix: Wrapped the bucket name in `.strip()` as a permanent safety net:
```python
  BUCKET_NAME = 'cloudvault-files-767397825366-ap-south-1-an'.strip()
```
- Verified the *exact* clean name afterward using AWS CLI instead of copy-pasting from 
  the console UI again:
```powershell
  aws s3 ls
```

**Bug 3 — `AccessDenied: ... not authorized to perform: s3:ListBucket`**
- Cause: The inline IAM policy I created for S3 access either wasn't saved to the correct 
  role, or wasn't attached at all — Lambda's auto-generated execution role 
  (`cloudvault-auto-expiry-role-<random-suffix>`) only had basic CloudWatch Logs permissions.
- Diagnosed using AWS CLI directly instead of guessing through the console:
```powershell
  aws iam list-role-policies --role-name cloudvault-auto-expiry-role-1e0w9tl7
  aws iam get-role-policy --role-name cloudvault-auto-expiry-role-1e0w9tl7 --policy-name cloudvault-s3-expiry-access
  aws lambda get-function-configuration --function-name cloudvault-auto-expiry --query "Role"
```
- Fix: Re-created the inline policy directly on the correct role, with the key detail that 
  `s3:ListBucket` needs the **bucket-level ARN** (no `/*`) while `s3:DeleteObject` needs 
  the **object-level ARN** (with `/*`):
```json
  {
    "Effect": "Allow",
    "Action": ["s3:ListBucket", "s3:DeleteObject"],
    "Resource": [
      "arn:aws:s3:::cloudvault-files-767397825366-ap-south-1-an",
      "arn:aws:s3:::cloudvault-files-767397825366-ap-south-1-an/*"
    ]
  }
```

---

### Concept Learned — AWS Cron Expression Fields
Six fields, always in this order: `Minutes  Hours  Day-of-month  Month  Day-of-week  Year`
- A specific number = "at exactly this value"; `*` = "every value"
- AWS requires **exactly one** of Day-of-month / Day-of-week to be `?` (since specifying 
  both as real values could contradict each other) — my schedule used `Day-of-month: *` 
  and `Day-of-week: ?`
- My schedule `0 * * * ? *` = "at minute 0, of every hour" → fires 24x/day, not once/day — 
  confirmed by checking CloudWatch logs for multiple invocations across the day, and by 
  running `aws scheduler get-schedule --name cloudvault-hourly-expiry-check`

### Key Takeaway
- Schedule frequency (how often the *check* runs) is separate from the expiry logic 
  (how old a file must be to qualify). Hourly checks + 24h expiry-in-code = tighter 
  cleanup than daily checks would give (worst case ~1hr delay vs. up to 48hrs delay)

### Reflection
- Tomorrow's focus: Day 6 — VPC + EC2 setup, Flask deployed on EC2

## Day 6 — 25-06-2026

**Status:** ✅ Cloud/Git

### Cloud/Git — VPC + EC2 Setup

#### What I Built
1. Created `cloudvault-vpc` (10.0.0.0/16) using the "VPC and more" wizard:
   - 2 Availability Zones, 2 public subnets, 2 private subnets
   - **NAT Gateway: explicitly set to "None"** — avoided a real cost trap (NAT Gateways 
     run ~$30-45/month even idle, and aren't Free Tier eligible)
   - Kept the default S3 Gateway VPC endpoint (free, allows private subnet → S3 access later)
2. Learned every AWS account also has a separate **Default VPC** (172.31.0.0/16) in each 
   region, auto-created on account setup — confirmed my EC2 instance launched into my 
   own `cloudvault-vpc`, not the default one (best practice: don't deploy into the default VPC)
3. Launched EC2 instance `cloudvault-web`:
   - AMI: Amazon Linux 2023 (not Amazon Linux 2 — that's about to hit end-of-life)
   - Instance type: **t3.micro** (corrected from the original plan's t2.micro — my account, 
     created after July 2025, is on the newer credit-based Free Tier model where t3.micro 
     is directly eligible, not just a t2.micro fallback)
   - Public subnet, auto-assign public IP enabled
   - Security group: SSH (22) restricted to My IP only, custom TCP 5000 also restricted to My IP
   - Created and downloaded key pair `cloudvault-key.pem`
4. Locked down key file permissions on Windows (required before SSH will trust the key):
```powershell
   icacls "C:\Users\sarat\.ssh\cloudvault-key.pem" /inheritance:r
   icacls "C:\Users\sarat\.ssh\cloudvault-key.pem" /grant:r "$($env:USERNAME):(R)"
```
5. SSH'd into the instance from PowerShell:
```powershell
   ssh -i "C:\Users\sarat\.ssh\cloudvault-key.pem" ec2-user@<public-ip>
```
6. Installed Flask on the EC2 instance itself (ran once prompt changed to `[ec2-user@...]$`):
```bash
   sudo dnf update -y
   sudo dnf install -y python3-pip
   pip3 install flask boto3 python-dotenv --user
```
7. Created GitHub Actions workflow skeleton at `.github/workflows/ci.yml` (placeholder — 
   full pipeline build comes Day 7/13)
8. Documented setup in `docs/vpc-ec2-setup.md`, committed:
```powershell
   git add .github\workflows\ci.yml docs\vpc-ec2-setup.md
   git commit -m "infra: add VPC and EC2 setup notes"
   git push origin main
```
9. **Stopped the EC2 instance** (Instance state → Stop) to avoid leaving it billing overnight

#### Key Takeaways
- `icacls /inheritance:r` strips inherited folder permissions; `/grant:r "$env:USERNAME:(R)"` 
  then grants read-only access to just my own Windows account — this combo is what makes 
  SSH trust a `.pem` file on Windows
- Once SSH'd in, the terminal prompt itself changes (`PS C:\Users\sarat>` → 
  `[ec2-user@ip-...]$`) — that prompt change is the signal you're now issuing commands 
  on the remote EC2 instance, not locally
- AWS Free Tier eligibility depends on **account creation date** — pre-July 2025 accounts 
  get t2.micro/t3.micro for 12 months; post-July 2025 accounts (mine) get a broader set 
  of instance types under a credit-based Free/Paid Plan model. Always trust the console's 
  "Free tier eligible" tag over older tutorials/PDFs.
- A stopped EC2 instance stops compute billing, but the attached EBS volume keeps 
  billing separately (within Free Tier's 30GB allowance, so not a concern yet — but 
  worth remembering for later)

### Reflection
- Tomorrow's focus: Day 7 — BST + Review Day (System Design Week 1 review, GitHub Actions CI completion)

## Day 8 — [Date]

**Status:** ✅ Cloud/Git

### Cloud/Git — RDS PostgreSQL Integration
- Identified true private subnets via route table inspection (not MapPublicIpOnLaunch, 
  unreliable since Day 6's manual IGW fix), used 1 private + 1 public subnet across 2 AZs 
  for the DB subnet group — RDS still fully isolated via "Public access: No" + security 
  group locked to only the EC2 SG
- Confirmed account is on the new credit-based Free Plan ($140 credit, 182 days) — no 
  "Free tier eligible" tag shown for RDS instance classes under this model; selected 
  db.t4g.micro manually
- Manually selected gp2 storage (gp3 isn't covered the same way)
- Created `file_uploads` table (id, filename, s3_key, upload_time, expiry_time), connected 
  via SSH tunnel through EC2 as a bastion host (RDS has no public access)
- Updated `app.py` so `/upload` logs metadata to RDS after each successful S3 upload
- Verified end-to-end with `check_db.py` — confirmed row insertion after a real upload

### Mistakes made / fixed (longest debugging chain so far)
1. RDS console renamed creation flow: "Express configuration" / "Full configuration" / 
   "Restore from S3" — picked Full configuration for manual control over storage type 
   and instance class
2. PostgreSQL 15+ on RDS sets `rds.force_ssl=1` by default — connections without SSL are 
   rejected. Fixed by adding `sslmode='require'` to every psycopg2.connect() call
3. **Root cause of a multi-hour "password authentication failed" chase:** the actual 
   RDS master username was `postgres`, not `cloudvault_admin` as assumed — likely the 
   username field reverted to default during creation and went unnoticed. PostgreSQL 
   deliberately returns the same generic "password authentication failed" error whether 
   the username or password is wrong (security best practice — doesn't leak which part 
   failed), which is why repeated password resets never fixed it
4. Diagnosed systematically rather than guessing further: tested via psql directly on 
   EC2 (ruling out local Windows/.env issues), confirmed PendingModifiedValues was empty 
   (ruling out a stuck password change), then queried AWS directly for the real username: