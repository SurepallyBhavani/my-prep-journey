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