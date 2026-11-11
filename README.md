# AWS CLI for Amazon S3 — Complete Hands-On Guide

A start-to-finish, copy-paste lab guide for working with **Amazon S3 from the AWS CLI**. Every section follows the same pattern: **concept → commands → verify → clean-up note**. Commands are written for **AWS CLI v2** on Linux/macOS (bash/zsh). Windows users: use WSL or Git Bash.

> Reference documentation
> - S3 User Guide: https://docs.aws.amazon.com/AmazonS3/latest/userguide/
> - CLI reference — high-level: https://docs.aws.amazon.com/cli/latest/reference/s3/index.html
> - CLI reference — low-level: https://docs.aws.amazon.com/cli/latest/reference/s3api/index.html
> - CLI reference — control plane: https://docs.aws.amazon.com/cli/latest/reference/s3control/index.html

> **Cost and safety:** Everything here fits inside normal Free-Tier-sized usage if you complete the clean-up (Section 29). Some labs (KMS key, replication, inventory, Batch Operations, S3 Express One Zone) can incur small charges. Never run these labs in a production account.

---

## Table of Contents

0. [Prerequisites and CLI setup](#0-prerequisites-and-cli-setup)
1. [Lab variables and sample data](#1-lab-variables-and-sample-data)
2. [The three CLI command sets: `s3`, `s3api`, `s3control`](#2-the-three-cli-command-sets)
3. [Buckets: create, inspect, list, delete](#3-buckets)
4. [Objects with `aws s3`: ls, cp, mv, rm](#4-objects-with-aws-s3)
5. [Filters: `--include` / `--exclude`, `--dryrun`](#5-filters-and-dry-runs)
6. [Sync](#6-sync)
7. [Streaming (stdin/stdout), metadata, content-type](#7-streaming-metadata-and-content-type)
8. [`s3api` object operations (put/get/head/copy/delete, pagination, JMESPath)](#8-s3api-object-operations)
9. [Multipart upload: automatic tuning and manual](#9-multipart-upload)
10. [Presigned URLs](#10-presigned-urls)
11. [Versioning and delete markers](#11-versioning)
12. [Storage classes, Glacier restore, Intelligent-Tiering](#12-storage-classes)
13. [Lifecycle rules](#13-lifecycle-rules)
14. [Encryption: SSE-S3, SSE-KMS, DSSE-KMS, SSE-C](#14-encryption)
15. [Block Public Access, bucket policies, ACLs, Object Ownership, CORS](#15-access-control)
16. [Access Points](#16-access-points)
17. [Static website hosting](#17-static-website-hosting)
18. [Replication (CRR/SRR)](#18-replication)
19. [Server access logging](#19-server-access-logging)
20. [Event notifications (SQS / EventBridge)](#20-event-notifications)
21. [Tagging](#21-tagging)
22. [Object Lock (WORM) and legal hold](#22-object-lock)
23. [Inventory and metrics](#23-inventory-and-metrics)
24. [Transfer Acceleration and Requester Pays](#24-transfer-acceleration-and-requester-pays)
25. [Checksums, object attributes, conditional writes](#25-checksums-attributes-conditional-writes)
26. [S3 Batch Operations](#26-s3-batch-operations)
27. [S3 Select (legacy note) and directory buckets (S3 Express One Zone)](#27-s3-select-and-directory-buckets)
28. [Performance tuning and CLI configuration](#28-performance-tuning-and-cli-configuration)
29. [Clean-up / teardown](#29-clean-up)
30. [Troubleshooting](#30-troubleshooting)
31. [Cheat sheet](#31-cheat-sheet)
32. [Capstone: end-to-end mini project](#32-capstone-end-to-end-project)

---

## 0. Prerequisites and CLI setup

### 0.1 Install AWS CLI v2

```bash
# Linux x86_64
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip -q awscliv2.zip && sudo ./aws/install
# macOS: brew install awscli      (or the official .pkg installer)
aws --version
```

Expected: `aws-cli/2.x.x ...`. Keep it current (`sudo ./aws/install --update`); several features used below (checksum flags, conditional writes) need a recent v2.

### 0.2 Configure credentials

Prefer **IAM Identity Center (SSO)** or a dedicated **IAM user/role with least privilege** over root keys.

```bash
aws configure                      # access key, secret, default region, output format
# or, for SSO:
aws configure sso
aws sso login --profile my-profile
export AWS_PROFILE=my-profile
```

Verify who you are:

```bash
aws sts get-caller-identity
aws configure list
```

### 0.3 Permissions for this lab

For a sandbox account, attach `AmazonS3FullAccess` plus `AWSKeyManagementServicePowerUser`, `AmazonSQSFullAccess`, and `IAMFullAccess` (only needed for Sections 14, 18, 20, 26). In real environments, scope down; a least-privilege example is in Section 15.5.

### 0.4 Handy tools

`jq` (optional), `curl`, `python3` (used in two labs), `openssl`, `md5sum` (macOS: `md5 -r`).

---

## 1. Lab variables and sample data

S3 bucket names are **globally unique across all AWS accounts**, so we embed the account ID.

```bash
export REGION=us-east-1          # primary region (change if you like)
export DR_REGION=us-west-2       # second region (replication lab)
export AWS_DEFAULT_REGION=$REGION
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

export BUCKET=s3lab-${ACCOUNT_ID}-main
export LOG_BUCKET=s3lab-${ACCOUNT_ID}-logs
export DEST_BUCKET=s3lab-${ACCOUNT_ID}-replica    # lives in $DR_REGION
export WEB_BUCKET=s3lab-${ACCOUNT_ID}-web
export LOCK_BUCKET=s3lab-${ACCOUNT_ID}-lock
echo $BUCKET
```

> New terminal later? Re-run this block. Tip: save it as `~/s3lab/env.sh` and `source` it.

Create the sample data:

```bash
mkdir -p ~/s3lab/data/{docs,images,logs} && cd ~/s3lab
echo "Hello S3 from the AWS CLI" > data/docs/hello.txt
echo "# Quarterly Report" > data/docs/report.md
printf 'id,name,amount\n1,alice,10\n2,bob,25\n3,carol,40\n' > data/docs/sales.csv
for i in 1 2 3; do echo "log line $i" > data/logs/app-$i.log; done
dd if=/dev/urandom of=data/images/photo.bin bs=1M count=3 2>/dev/null
cat > data/index.html <<'EOF'
<!doctype html><html><head><title>S3 Lab</title></head>
<body><h1>Hello from S3 static hosting</h1></body></html>
EOF
cat > data/error.html <<'EOF'
<!doctype html><html><body><h1>404 - not found</h1></body></html>
EOF
find data -type f | sort
```

---

## 2. The three CLI command sets

| Command set | Level | Use it for |
|---|---|---|
| `aws s3` | High-level | File-like operations: `ls cp mv rm sync mb rb presign website`. Handles recursion, parallelism, automatic multipart. |
| `aws s3api` | Low-level | 1:1 mapping to S3 REST API: every bucket/object configuration (versioning, policy, lifecycle, lock…). |
| `aws s3control` | Account/control-plane | Access Points, Batch Operations, Storage Lens, Multi-Region Access Points, account-level Block Public Access. |

Discover commands and options offline:

```bash
aws s3 help
aws s3 cp help
aws s3api help | head -50
aws s3api list-objects-v2 help
```

---

## 3. Buckets

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/creating-buckets-s3.html · naming rules: https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucketnamingrules.html

**Naming rules:** 3–63 chars, lowercase letters/numbers/hyphens (dots allowed but avoid them), start/end with a letter or number, globally unique.

### 3.1 Create

```bash
# High level (CLI sets LocationConstraint for you)
aws s3 mb s3://$BUCKET --region $REGION

# Low level: us-east-1 must NOT send a LocationConstraint
#   aws s3api create-bucket --bucket NAME --region us-east-1
# Any other region MUST:
aws s3api create-bucket --bucket $LOG_BUCKET --region $REGION
aws s3api create-bucket --bucket $DEST_BUCKET --region $DR_REGION \
  --create-bucket-configuration LocationConstraint=$DR_REGION
aws s3api create-bucket --bucket $WEB_BUCKET --region $REGION
```

New buckets are secure by default: **Block Public Access ON**, **ACLs disabled** (Object Ownership = BucketOwnerEnforced), and **default encryption SSE-S3**.

### 3.2 Inspect

```bash
aws s3 ls                                          # all buckets in the account
aws s3api list-buckets --query 'Buckets[].[Name,CreationDate]' --output table
aws s3api head-bucket --bucket $BUCKET             # silent = exists and you have access
aws s3api get-bucket-location --bucket $BUCKET     # null/None means us-east-1
aws s3api get-public-access-block --bucket $BUCKET
aws s3api get-bucket-ownership-controls --bucket $BUCKET
aws s3api get-bucket-encryption --bucket $BUCKET
```

### 3.3 Delete (preview)

```bash
aws s3 rb s3://$LOG_BUCKET            # fails if not empty (BucketNotEmpty)
aws s3 rb s3://$LOG_BUCKET --force    # empties current objects, then deletes
```
`--force` does **not** remove old versions or delete markers on versioned buckets. Use the `empty_bucket` function in Section 29.

> We delete `$LOG_BUCKET` here only to prove the command; recreate it:
> `aws s3 mb s3://$LOG_BUCKET --region $REGION`

---

## 4. Objects with `aws s3`

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/uploading-an-object-bucket.html

### 4.1 Upload

```bash
cd ~/s3lab
aws s3 cp data/docs/hello.txt s3://$BUCKET/docs/hello.txt          # single file
aws s3 cp data/ s3://$BUCKET/data/ --recursive                      # directory
```

### 4.2 List

```bash
aws s3 ls s3://$BUCKET                       # top-level prefixes ("PRE")
aws s3 ls s3://$BUCKET/data/docs/            # trailing slash = list inside the prefix
aws s3 ls s3://$BUCKET --recursive           # everything
aws s3 ls s3://$BUCKET --recursive --human-readable --summarize
```

Expected tail:
```
Total Objects: 9
   Total Size: 3.0 MiB
```

> Without a trailing slash, `aws s3 ls s3://$BUCKET/data/docs` lists things whose key *begins with* `data/docs`.

### 4.3 Download

```bash
mkdir -p ~/s3lab/download && cd ~/s3lab
aws s3 cp s3://$BUCKET/data/docs/hello.txt download/hello.txt
aws s3 cp s3://$BUCKET/data/ download/data/ --recursive
cat download/hello.txt
```

### 4.4 Copy between locations (server-side, no local download)

```bash
aws s3 cp s3://$BUCKET/data/docs/hello.txt s3://$BUCKET/backup/hello-copy.txt
aws s3 cp s3://$BUCKET/data/ s3://$BUCKET/backup/data/ --recursive
```

### 4.5 Move (copy + delete source)

```bash
aws s3 mv s3://$BUCKET/backup/hello-copy.txt s3://$BUCKET/archive/hello-moved.txt
aws s3 ls s3://$BUCKET/backup/ ; aws s3 ls s3://$BUCKET/archive/
```

### 4.6 Delete

```bash
aws s3 rm s3://$BUCKET/archive/hello-moved.txt
aws s3 rm s3://$BUCKET/backup/ --recursive
```
On a **versioned** bucket `rm` only adds a delete marker (Section 11).

### 4.7 Useful `cp` flags

| Flag | Purpose |
|---|---|
| `--recursive` | Operate on a whole prefix/directory |
| `--storage-class` | `STANDARD`, `STANDARD_IA`, `ONEZONE_IA`, `INTELLIGENT_TIERING`, `GLACIER_IR`, `GLACIER`, `DEEP_ARCHIVE`, `REDUCED_REDUNDANCY` |
| `--sse`, `--sse-kms-key-id`, `--sse-c`, `--sse-c-key` | Encryption (Section 14) |
| `--acl` | Canned ACL (`private`, `bucket-owner-full-control`, …; needs ACLs enabled) |
| `--metadata`, `--metadata-directive` | Custom user metadata |
| `--content-type`, `--cache-control`, `--content-disposition`, `--content-encoding`, `--content-language`, `--expires` | HTTP headers stored with the object |
| `--website-redirect` | Per-object redirect on website endpoints |
| `--no-guess-mime-type` | Don't auto-detect `Content-Type` |
| `--follow-symlinks` / `--no-follow-symlinks` | Symlink handling on upload |
| `--only-show-errors`, `--quiet`, `--no-progress` | Reduce output |
| `--dryrun` | Show what would happen |
| `--expected-size` | Needed when streaming from stdin and size > 50 GB |
| `--request-payer requester` | Access Requester-Pays buckets |
| `--checksum-algorithm`, `--checksum-mode` | Data-integrity checksums (Section 25) |
| `--force-glacier-transfer`, `--ignore-glacier-warnings` | Glacier-tier object handling |
| `--source-region` | Source region for cross-region copy |
| `--copy-props` | Which properties to copy on S3→S3 copy: `none`, `metadata-directive`, `default` |

---

## 5. Filters and dry-runs

`--exclude` and `--include` accept patterns (`*`, `?`, `[sequence]`, `[!sequence]`). **Filters are applied in order; later ones win.** Always start with `--dryrun`.

```bash
cd ~/s3lab
# Everything except logs
aws s3 cp data/ s3://$BUCKET/filtered/ --recursive --exclude "logs/*" --dryrun

# ONLY .txt and .md files (exclude all, then include)
aws s3 cp data/ s3://$BUCKET/filtered/ --recursive \
  --exclude "*" --include "*.txt" --include "*.md"

aws s3 ls s3://$BUCKET/filtered/ --recursive
```

Expected: only `docs/hello.txt` and `docs/report.md` appear.

> The wrong order, `--include "*.txt" --exclude "*"`, excludes everything.

Delete with filters:

```bash
aws s3 rm s3://$BUCKET/filtered/ --recursive --exclude "*" --include "*.md" --dryrun
aws s3 rm s3://$BUCKET/filtered/ --recursive --exclude "*" --include "*.md"
```

---

## 6. Sync

Sync copies only new/changed files (compares **size and last-modified time**). Direction matters: `sync SOURCE DEST`.

```bash
cd ~/s3lab
# Local -> S3
aws s3 sync data/ s3://$BUCKET/sync/
echo "new file" > data/docs/new.txt
aws s3 sync data/ s3://$BUCKET/sync/            # uploads only new.txt

# Mirror deletions: remove from S3 what no longer exists locally
rm data/docs/new.txt
aws s3 sync data/ s3://$BUCKET/sync/ --delete --dryrun
aws s3 sync data/ s3://$BUCKET/sync/ --delete

# S3 -> Local
mkdir -p restore && aws s3 sync s3://$BUCKET/sync/ restore/

# S3 -> S3 (also cross-bucket / cross-region)
aws s3 sync s3://$BUCKET/sync/ s3://$BUCKET/sync-copy/
```

Key options: `--delete`, `--exclude/--include`, `--size-only` (ignore timestamps), `--exact-timestamps` (S3→local: re-download if times differ), `--dryrun`, `--storage-class`, `--sse`, `--acl`.

Verify two trees match:

```bash
diff -r data restore && echo "IDENTICAL"
```

---

## 7. Streaming, metadata and content-type

### 7.1 Stdin/stdout with `-`

```bash
echo "streamed content" | aws s3 cp - s3://$BUCKET/stream/hello.txt
aws s3 cp s3://$BUCKET/stream/hello.txt -                      # prints to terminal

# Compress on the fly
tar czf - data/docs | aws s3 cp - s3://$BUCKET/stream/docs.tar.gz
aws s3 cp s3://$BUCKET/stream/docs.tar.gz - | tar tzf -        # list tar contents

# Streams > 50 GB need the size hint:
#   ... | aws s3 cp - s3://bucket/big.bin --expected-size 107374182400
```

### 7.2 Metadata and headers

```bash
aws s3 cp data/docs/report.md s3://$BUCKET/meta/report.md \
  --content-type "text/markdown" \
  --cache-control "max-age=3600" \
  --content-disposition 'attachment; filename="report.md"' \
  --metadata project=s3lab,owner=devops

aws s3api head-object --bucket $BUCKET --key meta/report.md
```

Expected (trimmed): `"ContentType": "text/markdown"`, `"CacheControl": "max-age=3600"`, `"Metadata": {"owner":"devops","project":"s3lab"}`.

S3 object metadata is **immutable**; to change it, copy the object onto itself with a replace directive:

```bash
aws s3 cp s3://$BUCKET/meta/report.md s3://$BUCKET/meta/report.md \
  --metadata-directive REPLACE --metadata project=s3lab-v2 \
  --content-type "text/markdown" --cache-control "max-age=60"
aws s3api head-object --bucket $BUCKET --key meta/report.md --query Metadata
```

---

## 8. `s3api` object operations

The low-level API is what you use for everything the high-level commands can't express.

### 8.1 put / get / head / copy / delete

```bash
cd ~/s3lab
aws s3api put-object --bucket $BUCKET --key api/hello.txt --body data/docs/hello.txt \
  --content-type text/plain --metadata origin=s3api
aws s3api head-object --bucket $BUCKET --key api/hello.txt
aws s3api get-object --bucket $BUCKET --key api/hello.txt download/api-hello.txt
cat download/api-hello.txt

# Partial (ranged) read: first 5 bytes
aws s3api get-object --bucket $BUCKET --key api/hello.txt --range bytes=0-4 download/part.txt
cat download/part.txt   # -> Hello

aws s3api copy-object --bucket $BUCKET --key api/hello-copy.txt \
  --copy-source $BUCKET/api/hello.txt

aws s3api delete-object --bucket $BUCKET --key api/hello-copy.txt
```

### 8.2 Listing with `list-objects-v2`, prefixes and delimiters

```bash
aws s3api list-objects-v2 --bucket $BUCKET --prefix data/ --delimiter / \
  --query '{Folders: CommonPrefixes[].Prefix, Files: Contents[].Key}'

aws s3api list-objects-v2 --bucket $BUCKET --prefix data/docs/ \
  --query 'Contents[].{Key:Key,Size:Size,Class:StorageClass}' --output table

# Largest 3 objects
aws s3api list-objects-v2 --bucket $BUCKET \
  --query 'reverse(sort_by(Contents,&Size))[:3].[Key,Size]' --output text

# Objects modified in the last day (JMESPath on a date string; format must match)
aws s3api list-objects-v2 --bucket $BUCKET \
  --query "Contents[?LastModified>='$(date -u -d '-1 day' +%Y-%m-%d 2>/dev/null || date -u -v-1d +%Y-%m-%d)'].Key"
```

### 8.3 Pagination

The CLI auto-paginates (1000 keys per request). Control it:

```bash
aws s3api list-objects-v2 --bucket $BUCKET --max-items 3          # returns NextToken if truncated
aws s3api list-objects-v2 --bucket $BUCKET --max-items 3 --starting-token <TOKEN>
aws s3api list-objects-v2 --bucket $BUCKET --page-size 100        # API page size (not total)
aws s3api list-objects-v2 --bucket $BUCKET --no-paginate --max-keys 5   # one raw page
aws s3api list-objects-v2 --bucket $BUCKET --start-after data/docs/hello.txt --max-items 3
```

### 8.4 Bulk delete (up to 1000 keys per call)

```bash
aws s3api list-objects-v2 --bucket $BUCKET --prefix filtered/ \
  --query '{Objects: Contents[].{Key:Key}}' --output json > /tmp/del.json
cat /tmp/del.json
aws s3api delete-objects --bucket $BUCKET --delete file:///tmp/del.json
```

(If the prefix is empty, `Objects` is `null` and the call errors — that just means nothing to delete.)

---

## 9. Multipart upload

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html

Rules: part size **5 MiB – 5 GiB** (last part can be smaller), up to **10,000 parts**, max object 5 TiB. Use multipart for objects > 100 MiB (required > 5 GiB for a single PUT).

### 9.1 Automatic (what `aws s3 cp/sync` already does)

The high-level commands switch to multipart at `multipart_threshold` (default 8 MB) using `multipart_chunksize` (default 8 MB).

```bash
aws configure set default.s3.multipart_threshold 16MB
aws configure set default.s3.multipart_chunksize 8MB
aws configure set default.s3.max_concurrent_requests 10

cd ~/s3lab
dd if=/dev/urandom of=big.bin bs=1M count=40 2>/dev/null
aws s3 cp big.bin s3://$BUCKET/mpu/auto-big.bin
aws s3api head-object --bucket $BUCKET --key mpu/auto-big.bin --query '[ContentLength,ETag]'
```

The ETag ends in `-N` (e.g. `"abc...-5"`), the number of parts — the signature of a multipart object.

### 9.2 Manual (full control with `s3api`)

```bash
cd ~/s3lab
dd if=/dev/urandom of=manual.bin bs=1M count=12 2>/dev/null
rm -f part-*; split -b 5M manual.bin part-         # part-aa (5M) part-ab (5M) part-ac (2M)
KEY=mpu/manual.bin

# 1) Initiate
UPLOAD_ID=$(aws s3api create-multipart-upload --bucket $BUCKET --key $KEY \
  --query UploadId --output text)
echo $UPLOAD_ID

# 2) Upload each part
N=1
for f in part-*; do
  aws s3api upload-part --bucket $BUCKET --key $KEY \
    --part-number $N --body $f --upload-id $UPLOAD_ID --query ETag --output text
  N=$((N+1))
done

# 3) Inspect what's been uploaded
aws s3api list-parts --bucket $BUCKET --key $KEY --upload-id $UPLOAD_ID \
  --query 'Parts[].[PartNumber,Size,ETag]' --output table
aws s3api list-multipart-uploads --bucket $BUCKET

# 4) Build the completion manifest straight from list-parts, then complete
aws s3api list-parts --bucket $BUCKET --key $KEY --upload-id $UPLOAD_ID \
  --query '{Parts: Parts[].{ETag: ETag, PartNumber: PartNumber}}' --output json > parts.json
aws s3api complete-multipart-upload --bucket $BUCKET --key $KEY \
  --upload-id $UPLOAD_ID --multipart-upload file://parts.json

# 5) Verify content is identical
aws s3 cp s3://$BUCKET/$KEY download/manual.bin
md5sum manual.bin download/manual.bin        # macOS: md5 -r ...
```

### 9.3 Abort an unfinished upload (stops storage charges for parts)

```bash
UP=$(aws s3api create-multipart-upload --bucket $BUCKET --key mpu/abandoned.bin \
  --query UploadId --output text)
aws s3api list-multipart-uploads --bucket $BUCKET --query 'Uploads[].[Key,UploadId]'
aws s3api abort-multipart-upload --bucket $BUCKET --key mpu/abandoned.bin --upload-id $UP
```

Incomplete uploads silently cost money. Always add the lifecycle rule `AbortIncompleteMultipartUpload` (Section 13).

### 9.4 Server-side copy of a huge object in parts

`upload-part-copy` copies byte ranges between objects without downloading:

```bash
UP=$(aws s3api create-multipart-upload --bucket $BUCKET --key mpu/copy-of-manual.bin \
  --query UploadId --output text)
SIZE=$(aws s3api head-object --bucket $BUCKET --key $KEY --query ContentLength --output text)
E1=$(aws s3api upload-part-copy --bucket $BUCKET --key mpu/copy-of-manual.bin \
  --upload-id $UP --part-number 1 --copy-source $BUCKET/$KEY \
  --copy-source-range bytes=0-5242879 --query CopyPartResult.ETag --output text)
E2=$(aws s3api upload-part-copy --bucket $BUCKET --key mpu/copy-of-manual.bin \
  --upload-id $UP --part-number 2 --copy-source $BUCKET/$KEY \
  --copy-source-range bytes=5242880-$((SIZE-1)) --query CopyPartResult.ETag --output text)
cat > copy-parts.json <<EOF
{"Parts":[{"ETag":$E1,"PartNumber":1},{"ETag":$E2,"PartNumber":2}]}
EOF
```
Note `$E1`/`$E2` already include the surrounding quotes, so the JSON above is valid. Then:
```bash
aws s3api complete-multipart-upload --bucket $BUCKET --key mpu/copy-of-manual.bin \
  --upload-id $UP --multipart-upload file://copy-parts.json
```
(Every part except the last must be ≥ 5 MiB; here part 1 is exactly 5 MiB and part 2 is the remainder.)

---

## 10. Presigned URLs

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html

A presigned URL grants **time-limited access using your credentials' permissions** without sharing keys.

```bash
aws s3 presign s3://$BUCKET/docs/hello.txt --expires-in 300       # 5 minutes
URL=$(aws s3 presign s3://$BUCKET/data/docs/hello.txt --expires-in 300)
curl -s "$URL"                                                    # -> Hello S3 from the AWS CLI
```

Test that the bucket is still private:

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://$BUCKET.s3.$REGION.amazonaws.com/data/docs/hello.txt   # 403
```

Notes:
- `aws s3 presign` only produces **GET** URLs. For presigned **PUT** (upload) use an SDK — Python/boto3 example:

```bash
pip install boto3 --quiet
python3 - <<EOF
import boto3
s3 = boto3.client("s3", region_name="$REGION")
url = s3.generate_presigned_url("put_object",
        Params={"Bucket": "$BUCKET", "Key": "presigned/upload.txt"}, ExpiresIn=300)
print(url)
EOF
```
Then upload with: `curl -X PUT --upload-file ~/s3lab/data/docs/hello.txt "<printed-url>"`.
- Max expiry: 7 days with IAM-user keys (`--expires-in 604800`); with temporary credentials (roles/SSO) the URL dies when the session token expires.
- Use SigV4 (default in CLI v2).

---

## 11. Versioning

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html

Versioning keeps every version of an object, protects against overwrites/deletes. Once enabled it can only be **suspended**, never fully disabled.

```bash
aws s3api put-bucket-versioning --bucket $BUCKET --versioning-configuration Status=Enabled
aws s3api get-bucket-versioning --bucket $BUCKET
```

### 11.1 Create versions

```bash
cd ~/s3lab
echo "version 1" > v.txt && aws s3 cp v.txt s3://$BUCKET/versioned/v.txt
echo "version 2" > v.txt && aws s3 cp v.txt s3://$BUCKET/versioned/v.txt
echo "version 3" > v.txt && aws s3 cp v.txt s3://$BUCKET/versioned/v.txt

aws s3api list-object-versions --bucket $BUCKET --prefix versioned/v.txt \
  --query 'Versions[].[VersionId,IsLatest,LastModified,Size]' --output table
```

### 11.2 Read an older version

```bash
V1=$(aws s3api list-object-versions --bucket $BUCKET --prefix versioned/v.txt \
  --query 'Versions[-1].VersionId' --output text)       # oldest is last
aws s3api get-object --bucket $BUCKET --key versioned/v.txt --version-id $V1 download/v1.txt
cat download/v1.txt                                      # -> version 1
```

### 11.3 Delete creates a delete marker; undelete by removing it

```bash
aws s3 rm s3://$BUCKET/versioned/v.txt
aws s3 ls s3://$BUCKET/versioned/                         # gone from normal listing
aws s3api list-object-versions --bucket $BUCKET --prefix versioned/v.txt \
  --query 'DeleteMarkers[].[VersionId,IsLatest]'

DM=$(aws s3api list-object-versions --bucket $BUCKET --prefix versioned/v.txt \
  --query 'DeleteMarkers[?IsLatest].VersionId' --output text)
aws s3api delete-object --bucket $BUCKET --key versioned/v.txt --version-id $DM   # removes marker
aws s3 ls s3://$BUCKET/versioned/                         # v.txt is back (version 3)
```

### 11.4 Permanently delete one specific version

```bash
aws s3api delete-object --bucket $BUCKET --key versioned/v.txt --version-id $V1
```

### 11.5 Restore an old version as the latest

```bash
aws s3api copy-object --bucket $BUCKET --key versioned/v.txt \
  --copy-source "$BUCKET/versioned/v.txt?versionId=$V2"       # V2: any older version id
```
(Server-side copy creates a **new** latest version with the old content.)

### 11.6 Suspend

```bash
aws s3api put-bucket-versioning --bucket $BUCKET --versioning-configuration Status=Suspended
aws s3api put-bucket-versioning --bucket $BUCKET --versioning-configuration Status=Enabled   # re-enable for the rest of the labs
```
MFA Delete (`--mfa-delete Enabled --mfa "serial token"`) can only be enabled by the **root user** — documented here for completeness; not part of the lab.

---

## 12. Storage classes

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html

| Class | CLI value | Typical use | Min duration |
|---|---|---|---|
| Standard | `STANDARD` | Frequent access | – |
| Intelligent-Tiering | `INTELLIGENT_TIERING` | Unknown/changing access | – |
| Standard-IA | `STANDARD_IA` | Infrequent, rapid access | 30 days |
| One Zone-IA | `ONEZONE_IA` | Re-creatable infrequent data (1 AZ) | 30 days |
| Glacier Instant Retrieval | `GLACIER_IR` | Archive, ms access | 90 days |
| Glacier Flexible Retrieval | `GLACIER` | Archive, minutes–hours | 90 days |
| Glacier Deep Archive | `DEEP_ARCHIVE` | Long-term archive, 12–48 h | 180 days |
| Express One Zone | directory buckets | Single-digit-ms (Section 27) | – |

### 12.1 Upload straight into a class

```bash
cd ~/s3lab
for c in STANDARD_IA INTELLIGENT_TIERING GLACIER_IR; do
  aws s3 cp data/docs/hello.txt s3://$BUCKET/classes/$c.txt --storage-class $c
done
aws s3api list-objects-v2 --bucket $BUCKET --prefix classes/ \
  --query 'Contents[].[Key,StorageClass]' --output table
```

### 12.2 Change class of an existing object (copy onto itself)

```bash
aws s3 cp s3://$BUCKET/data/docs/report.md s3://$BUCKET/data/docs/report.md --storage-class STANDARD_IA
```
For small objects, remember the 128 KB billing minimum on IA classes — use them for larger objects in real life.

### 12.3 Glacier Flexible / Deep Archive and restore

```bash
aws s3 cp data/docs/sales.csv s3://$BUCKET/archive/sales.csv --storage-class GLACIER
aws s3 cp s3://$BUCKET/archive/sales.csv download/sales.csv
# -> error: InvalidObjectState (object is archived)

aws s3api restore-object --bucket $BUCKET --key archive/sales.csv \
  --restore-request '{"Days":2,"GlacierJobParameters":{"Tier":"Standard"}}'

aws s3api head-object --bucket $BUCKET --key archive/sales.csv --query Restore
# -> "ongoing-request=\"true\""  (later: ongoing-request="false", expiry-date=...)
```

Tiers (Glacier Flexible): `Expedited` (1–5 min), `Standard` (3–5 h), `Bulk` (5–12 h). Deep Archive: `Standard` (12 h), `Bulk` (48 h). The restore creates a **temporary copy** for `Days` days; the object stays in Glacier. Once restored you can `cp` it normally. Glacier Instant Retrieval needs no restore.

### 12.4 Intelligent-Tiering archive tiers (opt-in)

```bash
cat > it.json <<'EOF'
{"Id":"archive-cold","Status":"Enabled",
 "Tierings":[{"Days":90,"AccessTier":"ARCHIVE_ACCESS"},{"Days":180,"AccessTier":"DEEP_ARCHIVE_ACCESS"}]}
EOF
aws s3api put-bucket-intelligent-tiering-configuration --bucket $BUCKET \
  --id archive-cold --intelligent-tiering-configuration file://it.json
aws s3api list-bucket-intelligent-tiering-configurations --bucket $BUCKET
aws s3api delete-bucket-intelligent-tiering-configuration --bucket $BUCKET --id archive-cold
```

---

## 13. Lifecycle rules

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html

Automate transitions between classes and expiration. Rules run asynchronously (roughly daily, at UTC midnight); you cannot "see" them act immediately.

```bash
cat > lifecycle.json <<'EOF'
{
  "Rules": [
    {
      "ID": "logs-tiering-and-expiry",
      "Status": "Enabled",
      "Filter": { "Prefix": "logs/" },
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 365 }
    },
    {
      "ID": "abort-incomplete-multipart",
      "Status": "Enabled",
      "Filter": {},
      "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 }
    },
    {
      "ID": "noncurrent-versions",
      "Status": "Enabled",
      "Filter": {},
      "NoncurrentVersionTransitions": [
        { "NoncurrentDays": 30, "StorageClass": "GLACIER" }
      ],
      "NoncurrentVersionExpiration": { "NoncurrentDays": 90 }
    },
    {
      "ID": "clean-expired-delete-markers",
      "Status": "Enabled",
      "Filter": {},
      "Expiration": { "ExpiredObjectDeleteMarker": true }
    },
    {
      "ID": "tmp-expire-7d",
      "Status": "Enabled",
      "Filter": { "Prefix": "tmp/" },
      "Expiration": { "Days": 7 }
    }
  ]
}
EOF
aws s3api put-bucket-lifecycle-configuration --bucket $BUCKET \
  --lifecycle-configuration file://lifecycle.json
aws s3api get-bucket-lifecycle-configuration --bucket $BUCKET \
  --query 'Rules[].[ID,Status]' --output table
```

Rules of thumb:
- `put-bucket-lifecycle-configuration` **replaces the whole config** — always send all rules.
- Transition to IA classes requires objects to be ≥ 30 days old; by default objects < 128 KB are not transitioned to IA/Glacier-IR.
- Filters can combine `Prefix`, `Tags`, `ObjectSizeGreaterThan`, `ObjectSizeLessThan` under `And`:

```json
"Filter": { "And": { "Prefix": "reports/", "Tags": [{"Key":"archive","Value":"yes"}], "ObjectSizeGreaterThan": 131072 } }
```

Remove all rules:

```bash
aws s3api delete-bucket-lifecycle --bucket $BUCKET
```

---

## 14. Encryption

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingEncryption.html

Since January 2023 **every new object is encrypted at rest with SSE-S3 by default**. You can choose a stronger/auditable option per object or as bucket default.

### 14.1 SSE-S3 (S3-managed keys, `AES256`)

```bash
cd ~/s3lab
aws s3 cp data/docs/hello.txt s3://$BUCKET/enc/sse-s3.txt --sse AES256
aws s3api head-object --bucket $BUCKET --key enc/sse-s3.txt --query ServerSideEncryption
# -> "AES256"
```

### 14.2 SSE-KMS (your KMS key — auditable via CloudTrail, key policy control)

```bash
KMS_KEY_ID=$(aws kms create-key --description "s3lab key" --query KeyMetadata.KeyId --output text)
aws kms create-alias --alias-name alias/s3lab --target-key-id $KMS_KEY_ID
KMS_ARN=$(aws kms describe-key --key-id $KMS_KEY_ID --query KeyMetadata.Arn --output text)

# Per object
aws s3 cp data/docs/hello.txt s3://$BUCKET/enc/sse-kms.txt \
  --sse aws:kms --sse-kms-key-id alias/s3lab
aws s3api head-object --bucket $BUCKET --key enc/sse-kms.txt \
  --query '[ServerSideEncryption,SSEKMSKeyId]'

# As the bucket default, with S3 Bucket Keys (cuts KMS request cost up to ~99%)
aws s3api put-bucket-encryption --bucket $BUCKET --server-side-encryption-configuration \
  "{\"Rules\":[{\"ApplyServerSideEncryptionByDefault\":{\"SSEAlgorithm\":\"aws:kms\",\"KMSMasterKeyID\":\"$KMS_ARN\"},\"BucketKeyEnabled\":true}]}"
aws s3api get-bucket-encryption --bucket $BUCKET

aws s3 cp data/docs/report.md s3://$BUCKET/enc/default-kms.md          # picks up bucket default
aws s3api head-object --bucket $BUCKET --key enc/default-kms.md \
  --query '[ServerSideEncryption,SSEKMSKeyId,BucketKeyEnabled]'
```

Your caller needs `kms:GenerateDataKey` and `kms:Decrypt` on the key. Cross-account readers need access via the key policy, too.

Revert the default to SSE-S3 so later labs are unaffected:

```bash
aws s3api put-bucket-encryption --bucket $BUCKET --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```

### 14.3 DSSE-KMS (dual-layer)

```bash
aws s3 cp data/docs/hello.txt s3://$BUCKET/enc/dsse.txt --sse aws:kms:dsse --sse-kms-key-id alias/s3lab
```

### 14.4 SSE-C (you supply the key on every request)

> **Important (2026 change):** since April 2026 SSE-C is **blocked by default** on new general purpose buckets (and existing buckets in accounts with no SSE-C objects). You must first explicitly unblock it with `PutBucketEncryption`. Prefer SSE-S3/SSE-KMS unless you truly need to hold keys yourself. Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/blocking-unblocking-s3-c-encryption-gpb.html

```bash
# Unblock SSE-C on this bucket (keeps SSE-S3 as the default algorithm)
aws s3api put-bucket-encryption --bucket $BUCKET --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"},"BlockedEncryptionTypes":{"EncryptionType":["NONE"]}}]}'

SSEC_KEY=$(openssl rand -hex 16)        # 32 ASCII chars = 256-bit key; store it safely!
cd ~/s3lab
aws s3 cp data/docs/hello.txt s3://$BUCKET/enc/ssec.txt --sse-c AES256 --sse-c-key $SSEC_KEY

aws s3 cp s3://$BUCKET/enc/ssec.txt - 2>&1 | head -2           # fails: key required
aws s3 cp s3://$BUCKET/enc/ssec.txt - --sse-c AES256 --sse-c-key $SSEC_KEY   # works
```

Lose the key = lose the data. To re-block SSE-C afterwards, put the configuration again with `"BlockedEncryptionTypes":{"EncryptionType":["SSE-C"]}`.

### 14.5 Enforce encryption/TLS through a bucket policy

See Section 15.2 (`DenyInsecureTransport`) and this deny-unencrypted-uploads example:

```bash
cat > enforce-kms.json <<EOF
{"Version":"2012-10-17","Statement":[{
 "Sid":"DenyNonKMSUploads","Effect":"Deny","Principal":"*","Action":"s3:PutObject",
 "Resource":"arn:aws:s3:::$BUCKET/secure/*",
 "Condition":{"StringNotEquals":{"s3:x-amz-server-side-encryption":"aws:kms"}}}]}
EOF
aws s3api put-bucket-policy --bucket $BUCKET --policy file://enforce-kms.json
aws s3 cp ~/s3lab/data/docs/hello.txt s3://$BUCKET/secure/a.txt                       # AccessDenied
aws s3 cp ~/s3lab/data/docs/hello.txt s3://$BUCKET/secure/a.txt --sse aws:kms --sse-kms-key-id alias/s3lab   # OK
aws s3api delete-bucket-policy --bucket $BUCKET
```

---

## 15. Access control

### 15.1 Block Public Access (BPA)

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html

Four independent switches at **bucket** and **account** level: `BlockPublicAcls`, `IgnorePublicAcls`, `BlockPublicPolicy`, `RestrictPublicBuckets`.

```bash
aws s3api get-public-access-block --bucket $BUCKET
aws s3api put-public-access-block --bucket $BUCKET --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Account-wide (s3control)
aws s3control get-public-access-block --account-id $ACCOUNT_ID
aws s3control put-public-access-block --account-id $ACCOUNT_ID --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

### 15.2 Bucket policies

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-policies.html

Example: **deny any non-TLS request** (a best-practice baseline).

```bash
cat > policy-tls.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyInsecureTransport",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": ["arn:aws:s3:::$BUCKET", "arn:aws:s3:::$BUCKET/*"],
    "Condition": { "Bool": { "aws:SecureTransport": "false" } }
  }]
}
EOF
aws s3api put-bucket-policy --bucket $BUCKET --policy file://policy-tls.json
aws s3api get-bucket-policy --bucket $BUCKET --query Policy --output text
aws s3api get-bucket-policy-status --bucket $BUCKET        # IsPublic: false
```

Test it (HTTP is rejected; the CLI uses HTTPS so normal commands still work):

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://$BUCKET.s3.$REGION.amazonaws.com/data/docs/hello.txt   # 403
```

Other common policy patterns:
- **Grant another account read access** — `"Principal": {"AWS": "arn:aws:iam::<OTHER_ACCOUNT>:root"}`, `"Action": ["s3:GetObject","s3:ListBucket"]` on bucket and `/*`.
- **Restrict to your VPC endpoint** — `"Condition": {"StringNotEquals": {"aws:SourceVpce": "vpce-xxxx"}}` with `Effect: Deny`.
- **Restrict to an IP range** — `"Condition": {"NotIpAddress": {"aws:SourceIp": "203.0.113.0/24"}}` with `Effect: Deny`.
- **Allow CloudFront OAC** — principal `cloudfront.amazonaws.com` + `AWS:SourceArn` of the distribution.

Remove:

```bash
aws s3api delete-bucket-policy --bucket $BUCKET
```

### 15.3 Object Ownership and ACLs

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/about-object-ownership.html

Modern practice: keep ACLs **disabled** (`BucketOwnerEnforced`) and use policies. Enable ACLs only for legacy needs (e.g., S3 log delivery older flow, cross-account writers).

```bash
aws s3api get-bucket-ownership-controls --bucket $BUCKET        # BucketOwnerEnforced

# Enable ACLs (bucket owner takes ownership when uploader grants full control)
aws s3api put-bucket-ownership-controls --bucket $BUCKET \
  --ownership-controls '{"Rules":[{"ObjectOwnership":"BucketOwnerPreferred"}]}'

aws s3 cp ~/s3lab/data/docs/hello.txt s3://$BUCKET/acl/hello.txt --acl bucket-owner-full-control
aws s3api get-object-acl --bucket $BUCKET --key acl/hello.txt
aws s3api get-bucket-acl --bucket $BUCKET

# Put it back to ACL-less
aws s3api put-bucket-ownership-controls --bucket $BUCKET \
  --ownership-controls '{"Rules":[{"ObjectOwnership":"BucketOwnerEnforced"}]}'
```

With `BucketOwnerEnforced`, any request with an ACL (other than bucket-owner-full-control) fails with `AccessControlListNotSupported`.

### 15.4 CORS

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/cors.html

Needed when browser JavaScript on one origin calls S3 on another.

```bash
cat > cors.json <<'EOF'
{"CORSRules":[{
  "AllowedOrigins":["https://example.com"],
  "AllowedMethods":["GET","PUT","POST"],
  "AllowedHeaders":["*"],
  "ExposeHeaders":["ETag"],
  "MaxAgeSeconds":3000}]}
EOF
aws s3api put-bucket-cors --bucket $BUCKET --cors-configuration file://cors.json
aws s3api get-bucket-cors --bucket $BUCKET

# Simulate a browser preflight
curl -s -D - -o /dev/null -X OPTIONS "https://$BUCKET.s3.$REGION.amazonaws.com/data/docs/hello.txt" \
  -H "Origin: https://example.com" -H "Access-Control-Request-Method: GET" | grep -i access-control

aws s3api delete-bucket-cors --bucket $BUCKET
```

### 15.5 Least-privilege IAM policy for a user/role (identity-based)

Allow a user to list only `app/` and read/write objects under it:

```bash
cat > iam-s3-app.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    { "Sid": "ListOnlyAppPrefix", "Effect": "Allow", "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::$BUCKET",
      "Condition": { "StringLike": { "s3:prefix": ["app/*", "app/"] } } },
    { "Sid": "ObjectRW", "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::$BUCKET/app/*" }
  ]
}
EOF
aws iam create-policy --policy-name s3lab-app-rw --policy-document file://iam-s3-app.json
aws iam delete-policy --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/s3lab-app-rw    # clean up when done
```

Remember: access = IAM policy **AND** bucket policy (same account: either allowing is enough unless an explicit Deny exists; cross-account: both must allow). An explicit Deny always wins.

---

## 16. Access Points

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-points.html

Named network endpoints with their own policy — one per application/team on a shared bucket.

```bash
aws s3control create-access-point --account-id $ACCOUNT_ID --name s3lab-ap --bucket $BUCKET
aws s3control list-access-points --account-id $ACCOUNT_ID --bucket $BUCKET
aws s3control get-access-point --account-id $ACCOUNT_ID --name s3lab-ap

AP_ARN=arn:aws:s3:$REGION:$ACCOUNT_ID:accesspoint/s3lab-ap

# Use the ARN anywhere a bucket name is accepted
aws s3 ls s3://$AP_ARN/data/docs/
aws s3api list-objects-v2 --bucket $AP_ARN --prefix data/docs/ --query 'Contents[].Key'
aws s3 cp s3://$AP_ARN/data/docs/hello.txt -
```

Give an access point a policy (this one allows only your own account's root principal to read `data/*`; in real use grant the specific role):

```bash
cat > ap-policy.json <<EOF
{"Version":"2012-10-17","Statement":[{
 "Effect":"Allow","Principal":{"AWS":"arn:aws:iam::$ACCOUNT_ID:root"},
 "Action":["s3:GetObject"],"Resource":"$AP_ARN/object/data/*"}]}
EOF
aws s3control put-access-point-policy --account-id $ACCOUNT_ID --name s3lab-ap --policy file://ap-policy.json
aws s3control get-access-point-policy --account-id $ACCOUNT_ID --name s3lab-ap
```

Restrict an access point to a VPC at creation with `--vpc-configuration VpcId=vpc-xxxx`. For the access point to delegate to its policies, the bucket policy must allow access via access points (condition `s3:DataAccessPointAccount`), otherwise bucket-level permissions still apply — in a single account with IAM permissions the commands above work as-is.

Delete:

```bash
aws s3control delete-access-point --account-id $ACCOUNT_ID --name s3lab-ap
```

Related: **Multi-Region Access Points** (`aws s3control create-multi-region-access-point`) give one global endpoint across buckets in several regions; **Object Lambda Access Points** transform data on read. Both are advanced and out of scope for this lab.

---

## 17. Static website hosting

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html

> This lab deliberately makes a bucket public. Use only the dedicated `$WEB_BUCKET` and tear it down afterwards. Production: put **CloudFront (HTTPS) + Origin Access Control** in front of a private bucket instead.

```bash
cd ~/s3lab
aws s3 cp data/index.html s3://$WEB_BUCKET/ --content-type text/html
aws s3 cp data/error.html s3://$WEB_BUCKET/ --content-type text/html

# Enable website hosting
aws s3 website s3://$WEB_BUCKET/ --index-document index.html --error-document error.html
aws s3api get-bucket-website --bucket $WEB_BUCKET

# Allow public reads: drop BPA for this bucket, then add a public-read policy
aws s3api put-public-access-block --bucket $WEB_BUCKET --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=false,RestrictPublicBuckets=false

cat > web-policy.json <<EOF
{"Version":"2012-10-17","Statement":[{
 "Sid":"PublicReadGetObject","Effect":"Allow","Principal":"*",
 "Action":"s3:GetObject","Resource":"arn:aws:s3:::$WEB_BUCKET/*"}]}
EOF
aws s3api put-bucket-policy --bucket $WEB_BUCKET --policy file://web-policy.json
aws s3api get-bucket-policy-status --bucket $WEB_BUCKET        # IsPublic: true
```

Test (HTTP only; the endpoint form is `bucket.s3-website-REGION.amazonaws.com` for us-east-1 and many older regions, and `bucket.s3-website.REGION.amazonaws.com` for newer regions such as us-east-2 — check the S3 endpoints table if you changed `$REGION`):

```bash
WEB_URL=http://$WEB_BUCKET.s3-website-$REGION.amazonaws.com
curl -s $WEB_URL                                       # index.html
curl -s -o /dev/null -w "%{http_code}\n" $WEB_URL/nope.html    # 404 (served from error.html)
```

Routing rules and redirects:

```bash
cat > website.json <<'EOF'
{"IndexDocument":{"Suffix":"index.html"},"ErrorDocument":{"Key":"error.html"},
 "RoutingRules":[{"Condition":{"KeyPrefixEquals":"old/"},
                  "Redirect":{"ReplaceKeyPrefixWith":"new/","HttpRedirectCode":"301"}}]}
EOF
aws s3api put-bucket-website --bucket $WEB_BUCKET --website-configuration file://website.json

# Redirect ALL requests to another host instead
#   --website-configuration '{"RedirectAllRequestsTo":{"HostName":"example.com","Protocol":"https"}}'

aws s3api delete-bucket-website --bucket $WEB_BUCKET      # disable hosting
```

Lock it back down when finished:

```bash
aws s3api delete-bucket-policy --bucket $WEB_BUCKET
aws s3api put-public-access-block --bucket $WEB_BUCKET --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

---

## 18. Replication

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html

- **CRR** = cross-region, **SRR** = same-region. Source **and** destination need **versioning**.
- Only **new** objects (after the rule exists) replicate automatically. Existing objects need **S3 Batch Replication** (Section 26 pattern).
- Replication is asynchronous (typically seconds to minutes; S3 RTC offers a 15-minute SLA).

### 18.1 Prepare

```bash
aws s3api put-bucket-versioning --bucket $BUCKET --versioning-configuration Status=Enabled
aws s3api put-bucket-versioning --bucket $DEST_BUCKET --region $DR_REGION \
  --versioning-configuration Status=Enabled
```

### 18.2 IAM role S3 assumes

```bash
cat > trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Principal":{"Service":"s3.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name s3lab-replication-role --assume-role-policy-document file://trust.json

cat > repl-perms.json <<EOF
{"Version":"2012-10-17","Statement":[
 {"Effect":"Allow","Action":["s3:GetReplicationConfiguration","s3:ListBucket"],
  "Resource":"arn:aws:s3:::$BUCKET"},
 {"Effect":"Allow","Action":["s3:GetObjectVersionForReplication","s3:GetObjectVersionAcl","s3:GetObjectVersionTagging"],
  "Resource":"arn:aws:s3:::$BUCKET/*"},
 {"Effect":"Allow","Action":["s3:ReplicateObject","s3:ReplicateDelete","s3:ReplicateTags"],
  "Resource":"arn:aws:s3:::$DEST_BUCKET/*"}]}
EOF
aws iam put-role-policy --role-name s3lab-replication-role \
  --policy-name s3lab-replication --policy-document file://repl-perms.json
ROLE_ARN=$(aws iam get-role --role-name s3lab-replication-role --query Role.Arn --output text)
sleep 10      # let IAM propagate
```

(If the source uses SSE-KMS objects, the role also needs `kms:Decrypt` on the source key and `kms:Encrypt`/`GenerateDataKey` on the destination key, and the rule needs `SourceSelectionCriteria` + `EncryptionConfiguration`.)

### 18.3 Replication rule

```bash
cat > replication.json <<EOF
{
  "Role": "$ROLE_ARN",
  "Rules": [{
    "ID": "replicate-all",
    "Status": "Enabled",
    "Priority": 1,
    "Filter": {},
    "DeleteMarkerReplication": { "Status": "Disabled" },
    "Destination": { "Bucket": "arn:aws:s3:::$DEST_BUCKET", "StorageClass": "STANDARD_IA" }
  }]
}
EOF
aws s3api put-bucket-replication --bucket $BUCKET --replication-configuration file://replication.json
aws s3api get-bucket-replication --bucket $BUCKET
```

### 18.4 Test

```bash
echo "replicate me $(date)" > ~/s3lab/repl.txt
aws s3 cp ~/s3lab/repl.txt s3://$BUCKET/replication/repl.txt
aws s3api head-object --bucket $BUCKET --key replication/repl.txt --query ReplicationStatus   # PENDING -> COMPLETED
sleep 30
aws s3 ls s3://$DEST_BUCKET/replication/ --region $DR_REGION
aws s3api head-object --bucket $DEST_BUCKET --key replication/repl.txt --region $DR_REGION \
  --query '[ReplicationStatus,StorageClass]'                  # REPLICA, STANDARD_IA
```

Filter by prefix/tag instead of `{}`: `"Filter": {"Prefix": "important/"}` or `"Filter": {"Tag": {"Key":"replicate","Value":"yes"}}` (use `And` to combine).

Disable: `aws s3api delete-bucket-replication --bucket $BUCKET`.

---

## 19. Server access logging

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/ServerLogs.html

Best-effort, delivered to a target bucket in the **same region**, typically within a few hours. For auditing API calls with guaranteed delivery use **CloudTrail data events**.

```bash
# Allow S3's logging service to write into the log bucket
cat > log-policy.json <<EOF
{"Version":"2012-10-17","Statement":[{
 "Sid":"S3ServerAccessLogsPolicy","Effect":"Allow",
 "Principal":{"Service":"logging.s3.amazonaws.com"},
 "Action":"s3:PutObject","Resource":"arn:aws:s3:::$LOG_BUCKET/access-logs/*",
 "Condition":{"ArnLike":{"aws:SourceArn":"arn:aws:s3:::$BUCKET"},
              "StringEquals":{"aws:SourceAccount":"$ACCOUNT_ID"}}}]}
EOF
aws s3api put-bucket-policy --bucket $LOG_BUCKET --policy file://log-policy.json

aws s3api put-bucket-logging --bucket $BUCKET --bucket-logging-status \
  "{\"LoggingEnabled\":{\"TargetBucket\":\"$LOG_BUCKET\",\"TargetPrefix\":\"access-logs/$BUCKET/\"}}"
aws s3api get-bucket-logging --bucket $BUCKET

# generate traffic, then wait (usually hours) and look
aws s3 ls s3://$BUCKET/ >/dev/null
aws s3 ls s3://$LOG_BUCKET/access-logs/ --recursive
```

Disable: `aws s3api put-bucket-logging --bucket $BUCKET --bucket-logging-status '{}'`.

CloudTrail data events for the bucket (separate from the above; creates cost):

```bash
# aws cloudtrail put-event-selectors --trail-name <trail> --advanced-event-selectors file://sel.json
```

---

## 20. Event notifications

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/EventNotifications.html

Destinations: **SQS**, **SNS**, **Lambda**, **EventBridge**. Events: `s3:ObjectCreated:*`, `s3:ObjectRemoved:*`, `s3:ObjectRestore:*`, `s3:Replication:*`, `s3:LifecycleExpiration:*`, `s3:LifecycleTransition`, `s3:IntelligentTiering`, `s3:ObjectTagging:*`, `s3:ObjectAcl:Put`.

> `put-bucket-notification-configuration` **replaces** the full configuration.

### 20.1 To SQS with prefix/suffix filter

```bash
Q_URL=$(aws sqs create-queue --queue-name s3lab-events --query QueueUrl --output text)
Q_ARN=$(aws sqs get-queue-attributes --queue-url $Q_URL --attribute-names QueueArn \
  --query Attributes.QueueArn --output text)

# Queue policy: allow only this bucket (from this account) to send
python3 - <<EOF
import json
policy = {"Version":"2012-10-17","Statement":[{
  "Effect":"Allow","Principal":{"Service":"s3.amazonaws.com"},
  "Action":"sqs:SendMessage","Resource":"$Q_ARN",
  "Condition":{"ArnLike":{"aws:SourceArn":"arn:aws:s3:::$BUCKET"},
               "StringEquals":{"aws:SourceAccount":"$ACCOUNT_ID"}}}]}
json.dump({"Policy": json.dumps(policy)}, open("sqs-attrs.json","w"))
EOF
aws sqs set-queue-attributes --queue-url $Q_URL --attributes file://sqs-attrs.json

cat > notif.json <<EOF
{"QueueConfigurations":[{
  "Id":"new-csv-uploads","QueueArn":"$Q_ARN",
  "Events":["s3:ObjectCreated:*"],
  "Filter":{"Key":{"FilterRules":[{"Name":"prefix","Value":"incoming/"},{"Name":"suffix","Value":".csv"}]}}}]}
EOF
aws s3api put-bucket-notification-configuration --bucket $BUCKET --notification-configuration file://notif.json
aws s3api get-bucket-notification-configuration --bucket $BUCKET
```

Test:

```bash
aws s3 cp ~/s3lab/data/docs/sales.csv s3://$BUCKET/incoming/sales.csv
aws s3 cp ~/s3lab/data/docs/hello.txt s3://$BUCKET/incoming/ignored.txt     # no event (suffix)
sleep 5
aws sqs receive-message --queue-url $Q_URL --wait-time-seconds 10 \
  --query 'Messages[0].Body' --output text | python3 -m json.tool | head -30
```
You'll see an `s3:ObjectCreated:Put` record with bucket name, key, size.

### 20.2 To EventBridge (richer filtering, many targets)

```bash
aws s3api put-bucket-notification-configuration --bucket $BUCKET \
  --notification-configuration '{"EventBridgeConfiguration":{}}'
```
Then create an EventBridge rule on source `aws.s3`, detail-type `Object Created`, with `detail.bucket.name = $BUCKET`.

### 20.3 Lambda / SNS

Same JSON structure using `LambdaFunctionConfigurations` (`LambdaFunctionArn`) or `TopicConfigurations` (`TopicArn`). Lambda needs a resource policy first:

```bash
# aws lambda add-permission --function-name FN --statement-id s3invoke --action lambda:InvokeFunction \
#   --principal s3.amazonaws.com --source-arn arn:aws:s3:::$BUCKET --source-account $ACCOUNT_ID
```

Clear all notifications: `aws s3api put-bucket-notification-configuration --bucket $BUCKET --notification-configuration '{}'`.

---

## 21. Tagging

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-tagging.html

Up to 10 tags per object. Tags drive lifecycle filters, replication filters, IAM conditions (`s3:ExistingObjectTag/<key>`), and cost allocation (bucket tags).

```bash
cd ~/s3lab
# At upload (URL-query format)
aws s3api put-object --bucket $BUCKET --key tags/a.txt --body data/docs/hello.txt \
  --tagging "env=dev&team=devops"

# On an existing object
aws s3api put-object-tagging --bucket $BUCKET --key tags/a.txt \
  --tagging '{"TagSet":[{"Key":"env","Value":"prod"},{"Key":"classification","Value":"internal"}]}'
aws s3api get-object-tagging --bucket $BUCKET --key tags/a.txt
aws s3api delete-object-tagging --bucket $BUCKET --key tags/a.txt

# Bucket tags (cost allocation)
aws s3api put-bucket-tagging --bucket $BUCKET \
  --tagging '{"TagSet":[{"Key":"Project","Value":"s3lab"},{"Key":"Owner","Value":"devops"}]}'
aws s3api get-bucket-tagging --bucket $BUCKET
```

`put-object-tagging` replaces the full tag set. S3 has no "find objects by tag" API — use Inventory/Athena or Resource Groups Tagging for buckets.

---

## 22. Object Lock

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html

WORM (write-once-read-many) protection. Modes:
- **GOVERNANCE** — protected, but users with `s3:BypassGovernanceRetention` can override with `--bypass-governance-retention`.
- **COMPLIANCE** — nobody (not even root) can delete/shorten until retention ends. **Don't use COMPLIANCE in a lab.**
- **Legal hold** — indefinite, independent of retention, toggled on/off.

```bash
# Must be enabled at creation (this also turns on versioning)
aws s3api create-bucket --bucket $LOCK_BUCKET --region $REGION --object-lock-enabled-for-bucket
aws s3api get-object-lock-configuration --bucket $LOCK_BUCKET

# Optional bucket default: 1 day GOVERNANCE
aws s3api put-object-lock-configuration --bucket $LOCK_BUCKET --object-lock-configuration \
  '{"ObjectLockEnabled":"Enabled","Rule":{"DefaultRetention":{"Mode":"GOVERNANCE","Days":1}}}'

# Per-object retention at upload
UNTIL=$(date -u -d '+1 day' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -v+1d +%Y-%m-%dT%H:%M:%SZ)
aws s3api put-object --bucket $LOCK_BUCKET --key locked/a.txt --body ~/s3lab/data/docs/hello.txt \
  --object-lock-mode GOVERNANCE --object-lock-retain-until-date $UNTIL
aws s3api get-object-retention --bucket $LOCK_BUCKET --key locked/a.txt

# Try to delete the specific version -> AccessDenied
VID=$(aws s3api list-object-versions --bucket $LOCK_BUCKET --prefix locked/a.txt --query 'Versions[0].VersionId' --output text)
aws s3api delete-object --bucket $LOCK_BUCKET --key locked/a.txt --version-id $VID

# Override (needs s3:BypassGovernanceRetention)
aws s3api delete-object --bucket $LOCK_BUCKET --key locked/a.txt --version-id $VID --bypass-governance-retention
```

Legal hold:

```bash
aws s3api put-object --bucket $LOCK_BUCKET --key locked/hold.txt --body ~/s3lab/data/docs/hello.txt
aws s3api put-object-legal-hold --bucket $LOCK_BUCKET --key locked/hold.txt --legal-hold Status=ON
aws s3api get-object-legal-hold --bucket $LOCK_BUCKET --key locked/hold.txt
aws s3api put-object-legal-hold --bucket $LOCK_BUCKET --key locked/hold.txt --legal-hold Status=OFF
```

Change retention: `aws s3api put-object-retention --bucket B --key K --retention '{"Mode":"GOVERNANCE","RetainUntilDate":"2030-01-01T00:00:00Z"}'` (lengthening is always allowed; shortening needs bypass in GOVERNANCE, impossible in COMPLIANCE).

Clean-up note: objects under retention can't be deleted until it expires (≤ 1 day here) unless you bypass — see Section 29.

---

## 23. Inventory and metrics

### 23.1 S3 Inventory (scheduled object listing as CSV/ORC/Parquet)

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-inventory.html

```bash
# Destination bucket policy (we reuse the log bucket under an inventory/ prefix)
cat > inv-policy.json <<EOF
{"Version":"2012-10-17","Statement":[
 {"Sid":"InventoryPolicy","Effect":"Allow","Principal":{"Service":"s3.amazonaws.com"},
  "Action":"s3:PutObject","Resource":"arn:aws:s3:::$LOG_BUCKET/inventory/*",
  "Condition":{"ArnLike":{"aws:SourceArn":"arn:aws:s3:::$BUCKET"},
               "StringEquals":{"aws:SourceAccount":"$ACCOUNT_ID","s3:x-amz-acl":"bucket-owner-full-control"}}},
 {"Sid":"S3ServerAccessLogsPolicy","Effect":"Allow","Principal":{"Service":"logging.s3.amazonaws.com"},
  "Action":"s3:PutObject","Resource":"arn:aws:s3:::$LOG_BUCKET/access-logs/*",
  "Condition":{"ArnLike":{"aws:SourceArn":"arn:aws:s3:::$BUCKET"},
               "StringEquals":{"aws:SourceAccount":"$ACCOUNT_ID"}}}]}
EOF
aws s3api put-bucket-policy --bucket $LOG_BUCKET --policy file://inv-policy.json

cat > inventory.json <<EOF
{"Id":"daily-all","IsEnabled":true,
 "Destination":{"S3BucketDestination":{"Bucket":"arn:aws:s3:::$LOG_BUCKET","Format":"CSV","Prefix":"inventory"}},
 "IncludedObjectVersions":"Current","Schedule":{"Frequency":"Daily"},
 "OptionalFields":["Size","LastModifiedDate","StorageClass","ETag","EncryptionStatus","IntelligentTieringAccessTier"]}
EOF
aws s3api put-bucket-inventory-configuration --bucket $BUCKET --id daily-all --inventory-configuration file://inventory.json
aws s3api list-bucket-inventory-configurations --bucket $BUCKET
```
(The policy above keeps the logging statement from Section 19 so both features work on the log bucket.) The first report arrives within ~48 h at `s3://$LOG_BUCKET/inventory/...`. Query with Athena, or feed it to Batch Operations as a manifest.

Delete: `aws s3api delete-bucket-inventory-configuration --bucket $BUCKET --id daily-all`.

### 23.2 Request metrics (CloudWatch) — extra cost

```bash
aws s3api put-bucket-metrics-configuration --bucket $BUCKET --id whole-bucket \
  --metrics-configuration '{"Id":"whole-bucket"}'
aws s3api put-bucket-metrics-configuration --bucket $BUCKET --id docs-only \
  --metrics-configuration '{"Id":"docs-only","Filter":{"Prefix":"data/docs/"}}'
aws s3api list-bucket-metrics-configurations --bucket $BUCKET
aws s3api delete-bucket-metrics-configuration --bucket $BUCKET --id docs-only
aws s3api delete-bucket-metrics-configuration --bucket $BUCKET --id whole-bucket
```

Free daily storage metrics are already in CloudWatch (`AWS/S3`: `BucketSizeBytes`, `NumberOfObjects`):

```bash
aws cloudwatch get-metric-statistics --namespace AWS/S3 --metric-name BucketSizeBytes \
  --dimensions Name=BucketName,Value=$BUCKET Name=StorageType,Value=StandardStorage \
  --start-time $(date -u -d '-3 days' +%FT%TZ 2>/dev/null || date -u -v-3d +%FT%TZ) \
  --end-time $(date -u +%FT%TZ) --period 86400 --statistics Average
```

Related (console/`s3control`): **Storage Lens** dashboards (`aws s3control put-storage-lens-configuration`) and **Storage Class Analysis** (`put-bucket-analytics-configuration`).

---

## 24. Transfer Acceleration and Requester Pays

### 24.1 Transfer Acceleration

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/transfer-acceleration.html

Uses CloudFront edge locations to speed long-distance transfers. Bucket name must be DNS-compatible and contain **no dots**.

```bash
aws s3api put-bucket-accelerate-configuration --bucket $BUCKET --accelerate-configuration Status=Enabled
aws s3api get-bucket-accelerate-configuration --bucket $BUCKET

# One-off
aws s3 cp ~/s3lab/big.bin s3://$BUCKET/accel/big.bin --endpoint-url https://s3-accelerate.amazonaws.com
# Permanent for the profile
aws configure set default.s3.use_accelerate_endpoint true
aws s3 cp ~/s3lab/big.bin s3://$BUCKET/accel/big2.bin
aws configure set default.s3.use_accelerate_endpoint false        # turn back off

aws s3api put-bucket-accelerate-configuration --bucket $BUCKET --accelerate-configuration Status=Suspended
```
You pay an extra per-GB fee only when acceleration actually speeds up the transfer.

### 24.2 Requester Pays

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/RequesterPaysBuckets.html

The *requester* is billed for requests and data transfer (requests must be authenticated and include the flag).

```bash
aws s3api put-bucket-request-payment --bucket $BUCKET --request-payment-configuration Payer=Requester
aws s3api get-bucket-request-payment --bucket $BUCKET

aws s3 cp s3://$BUCKET/data/docs/hello.txt - --request-payer requester
aws s3api get-object --bucket $BUCKET --key data/docs/hello.txt --request-payer requester download/rp.txt

aws s3api put-bucket-request-payment --bucket $BUCKET --request-payment-configuration Payer=BucketOwner
```

### 24.3 Reading public/open-data buckets without credentials

```bash
aws s3 ls s3://noaa-gsod-pds/ --no-sign-request --region us-east-1 | head
```
`--no-sign-request` sends anonymous requests (works only on publicly readable buckets).

---

## 25. Checksums, attributes, conditional writes

### 25.1 Checksums (integrity validation)

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/checking-object-integrity.html

Algorithms: `CRC64NVME`, `CRC32`, `CRC32C`, `SHA1`, `SHA256`. Recent CLI v2 versions compute a default checksum automatically on uploads; you can choose explicitly:

```bash
cd ~/s3lab
aws s3 cp data/docs/hello.txt s3://$BUCKET/checksum/hello.txt --checksum-algorithm SHA256
aws s3api head-object --bucket $BUCKET --key checksum/hello.txt --checksum-mode ENABLED \
  --query '[ChecksumSHA256,ContentLength]'

# Independently verify locally (SHA-256, base64-encoded as S3 reports it)
openssl dgst -sha256 -binary data/docs/hello.txt | base64

# Verify on download
aws s3 cp s3://$BUCKET/checksum/hello.txt download/hello-verified.txt --checksum-mode ENABLED
```

The two base64 strings must match.

### 25.2 `get-object-attributes` (metadata without downloading)

```bash
aws s3api get-object-attributes --bucket $BUCKET --key mpu/manual.bin \
  --object-attributes ETag Checksum ObjectParts StorageClass ObjectSize
```
For multipart objects, `ObjectParts` lists part count and sizes.

### 25.3 Conditional writes (prevent overwrite / lost updates)

`If-None-Match: *` makes a write succeed only if the key does **not** exist (needs a recent CLI v2; older ones lack the flag).

```bash
echo "first" > cond.txt
aws s3api put-object --bucket $BUCKET --key cond/lock.txt --body cond.txt --if-none-match "*"   # succeeds
aws s3api put-object --bucket $BUCKET --key cond/lock.txt --body cond.txt --if-none-match "*"   # 412 PreconditionFailed
```

`--if-match <etag>` on `put-object`/`complete-multipart-upload` gives optimistic concurrency; `get-object` also accepts `--if-match`, `--if-none-match`, `--if-modified-since`, `--if-unmodified-since`.

---

## 26. S3 Batch Operations

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/batch-ops.html

Run one action (copy, tag, ACL, restore, Lambda invoke, Object Lock, replicate, checksum compute…) over millions of objects, driven by a **manifest** (CSV or S3 Inventory report) via `s3control`. Lab: tag all objects under `batch/` as `processed=true`.

```bash
cd ~/s3lab
# 1) Objects to process
for i in 1 2 3; do echo "obj $i" | aws s3 cp - s3://$BUCKET/batch/obj-$i.txt; done

# 2) CSV manifest: bucket,key (keys URL-encoded; ours are plain)
for i in 1 2 3; do echo "$BUCKET,batch/obj-$i.txt"; done > manifest.csv
aws s3 cp manifest.csv s3://$BUCKET/manifests/manifest.csv
MANIFEST_ETAG=$(aws s3api head-object --bucket $BUCKET --key manifests/manifest.csv \
  --query ETag --output text | tr -d '"')

# 3) IAM role for Batch Operations
cat > bo-trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Principal":{"Service":"batchoperations.s3.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name s3lab-batch-role --assume-role-policy-document file://bo-trust.json
cat > bo-perms.json <<EOF
{"Version":"2012-10-17","Statement":[
 {"Effect":"Allow","Action":["s3:PutObjectTagging","s3:PutObjectVersionTagging"],"Resource":"arn:aws:s3:::$BUCKET/batch/*"},
 {"Effect":"Allow","Action":["s3:GetObject","s3:GetObjectVersion"],"Resource":"arn:aws:s3:::$BUCKET/manifests/*"},
 {"Effect":"Allow","Action":["s3:PutObject"],"Resource":"arn:aws:s3:::$BUCKET/batch-reports/*"}]}
EOF
aws iam put-role-policy --role-name s3lab-batch-role --policy-name s3lab-batch --policy-document file://bo-perms.json
BO_ROLE=$(aws iam get-role --role-name s3lab-batch-role --query Role.Arn --output text)
sleep 10

# 4) Create the job
JOB_ID=$(aws s3control create-job --account-id $ACCOUNT_ID --no-confirmation-required --priority 10 \
  --role-arn $BO_ROLE \
  --operation '{"S3PutObjectTagging":{"TagSet":[{"Key":"processed","Value":"true"}]}}' \
  --manifest "{\"Spec\":{\"Format\":\"S3BatchOperations_CSV_20180820\",\"Fields\":[\"Bucket\",\"Key\"]},\"Location\":{\"ObjectArn\":\"arn:aws:s3:::$BUCKET/manifests/manifest.csv\",\"ETag\":\"$MANIFEST_ETAG\"}}" \
  --report "{\"Bucket\":\"arn:aws:s3:::$BUCKET\",\"Prefix\":\"batch-reports\",\"Format\":\"Report_CSV_20180820\",\"Enabled\":true,\"ReportScope\":\"AllTasks\"}" \
  --query JobId --output text)
echo $JOB_ID

# 5) Watch it
aws s3control describe-job --account-id $ACCOUNT_ID --job-id $JOB_ID \
  --query 'Job.[Status,ProgressSummary]'
aws s3control list-jobs --account-id $ACCOUNT_ID --job-statuses Active Complete Failed

# 6) Verify
aws s3api get-object-tagging --bucket $BUCKET --key batch/obj-1.txt
```

Status moves `New → Preparing → Ready → Active → Complete`. Controls: `update-job-status --requested-job-status Cancelled`, `update-job-priority`. For **existing-object replication** use the `S3ReplicateObject` operation with a manifest generated by the replication configuration (`--manifest-generator`).

---

## 27. S3 Select and directory buckets

### 27.1 S3 Select (legacy)

`select-object-content` runs SQL against a single CSV/JSON/Parquet object server-side. **AWS stopped offering S3 Select to new customers in July 2024**; it works only for accounts already using it. For new work use **Athena** (or load the data and filter locally). Syntax for reference:

```bash
aws s3api select-object-content --bucket $BUCKET --key data/docs/sales.csv \
  --expression "SELECT s.name, s.amount FROM S3Object s WHERE CAST(s.amount AS INT) > 15" \
  --expression-type SQL \
  --input-serialization '{"CSV":{"FileHeaderInfo":"USE"}}' \
  --output-serialization '{"JSON":{}}' select-out.json && cat select-out.json
```

### 27.2 Directory buckets — S3 Express One Zone

Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/directory-buckets-overview.html

Single-AZ, single-digit-millisecond storage for latency-sensitive workloads. Names must end with `--<az-id>--x-s3`. Not every AZ supports it; if creation fails, try another AZ ID.

```bash
AZ_ID=$(aws ec2 describe-availability-zones --region $REGION \
  --query 'AvailabilityZones[0].ZoneId' --output text)         # e.g. use1-az1
EXPRESS=s3lab-${ACCOUNT_ID}--${AZ_ID}--x-s3
aws s3api create-bucket --bucket $EXPRESS --region $REGION \
  --create-bucket-configuration "Location={Type=AvailabilityZone,Name=$AZ_ID},Bucket={DataRedundancy=SingleAvailabilityZone,Type=Directory}"

aws s3 cp ~/s3lab/data/docs/hello.txt s3://$EXPRESS/hello.txt
aws s3 ls s3://$EXPRESS/
aws s3api list-directory-buckets

aws s3 rm s3://$EXPRESS/hello.txt
aws s3api delete-bucket --bucket $EXPRESS --region $REGION
```
Differences: only `/` delimiter, session-based auth (handled by CLI), no versioning/replication/Object Lock/website hosting, lifecycle supports expiration only.

---

## 28. Performance tuning and CLI configuration

Docs: https://docs.aws.amazon.com/cli/latest/topic/s3-config.html

### 28.1 `s3` settings (`~/.aws/config`)

```ini
[default]
region = us-east-1
s3 =
  max_concurrent_requests = 20
  max_queue_size = 10000
  multipart_threshold = 64MB
  multipart_chunksize = 16MB
  max_bandwidth = 100MB/s
  use_accelerate_endpoint = false
  addressing_style = auto
  payload_signing_enabled = false
```

Same via CLI: `aws configure set default.s3.max_concurrent_requests 20`. Use a profile name instead of `default` for per-profile tuning (`profile.NAME.s3.max_concurrent_requests`).

| Setting | Effect |
|---|---|
| `max_concurrent_requests` (10) | Parallel transfer threads — raise for many small files / fat pipes |
| `max_queue_size` (1000) | Tasks queued |
| `multipart_threshold` (8MB) / `multipart_chunksize` (8MB) | When and how big parts are |
| `max_bandwidth` | Throttle (be a good neighbour) |
| `addressing_style` | `path`, `virtual`, `auto` (needed for some S3-compatible endpoints) |
| `use_accelerate_endpoint` | Transfer Acceleration |

### 28.2 Practical tips

- Many small files → raise `max_concurrent_requests`; or tar them first.
- Parallelize across prefixes: S3 supports ≥ 3,500 PUT/COPY/POST/DELETE and ≥ 5,500 GET/HEAD requests per second **per prefix**.
- Run from an EC2 instance in the same region plus a **Gateway VPC endpoint** for S3 (free) to avoid NAT costs:

```bash
aws ec2 create-vpc-endpoint --vpc-id vpc-xxxx --service-name com.amazonaws.$REGION.s3 \
  --route-table-ids rtb-xxxx
```
- `--only-show-errors` makes cron-driven syncs quiet; `--no-progress` for CI logs.
- Retries/timeouts: `AWS_MAX_ATTEMPTS=10`, `AWS_RETRY_MODE=adaptive`, `--cli-read-timeout 0 --cli-connect-timeout 60` for slow links.

### 28.3 Output control and debugging

```bash
aws s3api list-buckets --output json|text|table|yaml
aws s3 ls --debug 2>&1 | head -40         # full request/response trace
aws s3 cp x s3://b/x --endpoint-url https://s3.us-west-2.amazonaws.com    # explicit endpoint
aws s3 cp x s3://b/x --profile other --region eu-west-1
```

### 28.4 S3-compatible storage (MinIO, Ceph, etc.)

```bash
aws s3 ls --endpoint-url http://localhost:9000 --profile minio
aws configure set s3.addressing_style path --profile minio
```

---

## 29. Clean-up

Run in this order to stop all charges. Re-export the Section 1 variables first if you opened a new shell.

```bash
# --- helper: empty a bucket including all versions and delete markers ---
empty_bucket() {
  local b="$1" region="${2:-$REGION}"
  aws s3 rm "s3://$b" --recursive --region "$region" >/dev/null 2>&1 || true
  for kind in Versions DeleteMarkers; do
    while : ; do
      aws s3api list-object-versions --bucket "$b" --region "$region" --max-items 1000 \
        --query "{Objects: ${kind}[].{Key:Key,VersionId:VersionId}}" --output json > /tmp/del.json
      grep -q '"Objects": null' /tmp/del.json && break
      aws s3api delete-objects --bucket "$b" --region "$region" \
        --delete file:///tmp/del.json --bypass-governance-retention >/dev/null
    done
  done
  # abort any in-flight multipart uploads
  for id in $(aws s3api list-multipart-uploads --bucket "$b" --region "$region" \
      --query 'Uploads[].[Key,UploadId]' --output text 2>/dev/null | tr '\t' ','); do
    aws s3api abort-multipart-upload --bucket "$b" --region "$region" \
      --key "${id%%,*}" --upload-id "${id##*,}"
  done
}
# (If delete-objects with --bypass-governance-retention is denied for lock-bucket objects,
#  wait for retention to expire (<= 1 day) and re-run.)

# 1. Turn off features that write into/depend on buckets
aws s3api delete-bucket-replication --bucket $BUCKET 2>/dev/null
aws s3api put-bucket-logging --bucket $BUCKET --bucket-logging-status '{}'
aws s3api put-bucket-notification-configuration --bucket $BUCKET --notification-configuration '{}'
aws s3api delete-bucket-inventory-configuration --bucket $BUCKET --id daily-all 2>/dev/null
aws s3api delete-bucket-lifecycle --bucket $BUCKET 2>/dev/null
aws s3api delete-bucket-policy --bucket $BUCKET 2>/dev/null
aws s3control delete-access-point --account-id $ACCOUNT_ID --name s3lab-ap 2>/dev/null

# 2. Empty and delete every lab bucket
empty_bucket $BUCKET        && aws s3 rb s3://$BUCKET
empty_bucket $LOG_BUCKET    && aws s3 rb s3://$LOG_BUCKET
empty_bucket $WEB_BUCKET    && aws s3 rb s3://$WEB_BUCKET
empty_bucket $LOCK_BUCKET   && aws s3 rb s3://$LOCK_BUCKET
empty_bucket $DEST_BUCKET $DR_REGION && aws s3 rb s3://$DEST_BUCKET --region $DR_REGION

# 3. Other resources
aws sqs delete-queue --queue-url $Q_URL
aws iam delete-role-policy --role-name s3lab-replication-role --policy-name s3lab-replication
aws iam delete-role --role-name s3lab-replication-role
aws iam delete-role-policy --role-name s3lab-batch-role --policy-name s3lab-batch
aws iam delete-role --role-name s3lab-batch-role
aws kms delete-alias --alias-name alias/s3lab
aws kms schedule-key-deletion --key-id $KMS_KEY_ID --pending-window-in-days 7

# 4. Restore CLI defaults and delete local files
aws configure set default.s3.use_accelerate_endpoint false
rm -rf ~/s3lab
```

(Using `rb --force` is fine for non-versioned buckets, but the helper above is safe for every bucket type.)

---

## 30. Troubleshooting

| Error / symptom | Cause | Fix |
|---|---|---|
| `Unable to locate credentials` | No profile/env creds | `aws configure`, `aws sso login`, or set `AWS_PROFILE` |
| `AccessDenied` / `403` | IAM policy, bucket policy, BPA, SCP, KMS key policy, or object owned by another account | `aws sts get-caller-identity`; check explicit Deny; check `kms:Decrypt`; use IAM Policy Simulator |
| `NoSuchBucket` | Typo / wrong region / deleted | `aws s3 ls`; `aws s3api head-bucket` |
| `BucketAlreadyExists` | Name taken by another account | Add a unique suffix (account ID) |
| `BucketAlreadyOwnedByYou` | You already created it (us-east-1 returns 200 instead) | Safe to ignore |
| `IllegalLocationConstraintException` | `--region` ≠ `LocationConstraint`, or LocationConstraint sent for us-east-1 | Match them; omit for us-east-1 |
| `PermanentRedirect` / `301` | Wrong regional endpoint | `aws s3api get-bucket-location`; pass `--region` |
| `BucketNotEmpty` | Objects/versions/markers/MPUs remain | Use `empty_bucket` (Section 29) |
| `SignatureDoesNotMatch` / `RequestTimeTooSkewed` | Wrong secret key, clock drift > 15 min | Fix keys; sync NTP |
| `SlowDown` / `503` | Request rate too high on a prefix | Back off, `AWS_RETRY_MODE=adaptive`, spread over prefixes |
| `InvalidObjectState` | Object is in Glacier/Deep Archive | `restore-object` and wait |
| `AccessControlListNotSupported` | ACLs disabled (BucketOwnerEnforced) | Use policies or enable `BucketOwnerPreferred` |
| `InvalidRequest` mentioning SSE-C | SSE-C is blocked on the bucket | Unblock via `put-bucket-encryption` (Section 14.4) |
| `MalformedXML` / `InvalidArgument` on lifecycle | Missing `Filter`, conflicting actions in one `Expiration` | Validate JSON, use one action type per `Expiration` object |
| `NoSuchBucketPolicy`, `NoSuchLifecycleConfiguration`, `NoSuchCORSConfiguration`… | Nothing configured | Expected; not an error in practice |
| `PreconditionFailed` (412) | Conditional write/read check failed | Intended with `--if-none-match`/`--if-match` |
| Presigned URL → `ExpiredToken`/`Request has expired` | URL or session expired | Regenerate; use long-lived creds for long URLs |
| `sync` re-uploads everything | Timestamps changed | Add `--size-only` |
| Weird results from `ls` | Missing trailing `/` on prefix | `s3://b/dir/` |
| Uploading from a shell: `Argument list too long` / globbing | Shell expansion | Quote patterns: `--include "*.log"` |

Quick diagnostics:

```bash
aws s3 ls s3://$BUCKET --debug 2>&1 | grep -iE "region|endpoint|status|x-amz-request-id"
aws s3api get-bucket-location --bucket $BUCKET
aws s3api get-bucket-policy-status --bucket $BUCKET
aws iam simulate-principal-policy --policy-source-arn $(aws sts get-caller-identity --query Arn --output text) \
  --action-names s3:GetObject --resource-arns arn:aws:s3:::$BUCKET/data/docs/hello.txt   # for IAM users/roles
```

---

## 31. Cheat sheet

```bash
# ---------- buckets ----------
aws s3 mb s3://B                    aws s3 rb s3://B [--force]
aws s3 ls                           aws s3api head-bucket --bucket B
# ---------- objects ----------
aws s3 ls s3://B/prefix/ --recursive --human-readable --summarize
aws s3 cp SRC DST [--recursive --exclude "*" --include "*.x" --dryrun]
aws s3 mv SRC DST                   aws s3 rm s3://B/k [--recursive]
aws s3 sync SRC DST [--delete --size-only --exclude P --dryrun]
echo hi | aws s3 cp - s3://B/k      aws s3 cp s3://B/k -
aws s3 presign s3://B/k --expires-in 3600
# ---------- config ----------
aws s3api put-bucket-versioning --bucket B --versioning-configuration Status=Enabled
aws s3api list-object-versions --bucket B --prefix k
aws s3api put-bucket-lifecycle-configuration --bucket B --lifecycle-configuration file://l.json
aws s3api put-bucket-policy --bucket B --policy file://p.json
aws s3api put-public-access-block --bucket B --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
aws s3api put-bucket-encryption --bucket B --server-side-encryption-configuration file://e.json
aws s3api put-bucket-replication --bucket B --replication-configuration file://r.json
aws s3api put-bucket-notification-configuration --bucket B --notification-configuration file://n.json
aws s3api put-bucket-cors --bucket B --cors-configuration file://c.json
aws s3api put-bucket-logging --bucket B --bucket-logging-status file://log.json
aws s3api put-bucket-tagging --bucket B --tagging file://t.json
aws s3api put-object-lock-configuration --bucket B --object-lock-configuration file://ol.json
aws s3api put-bucket-accelerate-configuration --bucket B --accelerate-configuration Status=Enabled
# ---------- objects (api) ----------
aws s3api head-object|get-object|put-object|copy-object|delete-object|delete-objects
aws s3api restore-object --bucket B --key K --restore-request '{"Days":2,"GlacierJobParameters":{"Tier":"Standard"}}'
aws s3api create-multipart-upload | upload-part | complete-multipart-upload | abort-multipart-upload
# ---------- control plane ----------
aws s3control create-access-point|create-job|put-public-access-block|put-storage-lens-configuration
# ---------- JMESPath ----------
--query 'Contents[?Size>`1000000`].[Key,Size]' --output table
--query 'reverse(sort_by(Contents,&LastModified))[:5].Key'
```

---

## 32. Capstone: end-to-end project

Goal: a secure, versioned, encrypted, cost-optimised **application artifact bucket** with notifications — built only with the CLI, in one script. Re-run the Section 1 variables first.

```bash
#!/usr/bin/env bash
set -euo pipefail
B=s3lab-${ACCOUNT_ID}-capstone

# 1. Bucket, security baseline
aws s3api create-bucket --bucket $B --region $REGION
aws s3api put-public-access-block --bucket $B --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
aws s3api put-bucket-ownership-controls --bucket $B \
  --ownership-controls '{"Rules":[{"ObjectOwnership":"BucketOwnerEnforced"}]}'
aws s3api put-bucket-encryption --bucket $B --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"},"BucketKeyEnabled":true}]}'

# 2. TLS-only policy
cat > /tmp/cap-policy.json <<EOF
{"Version":"2012-10-17","Statement":[{"Sid":"TLSOnly","Effect":"Deny","Principal":"*","Action":"s3:*",
 "Resource":["arn:aws:s3:::$B","arn:aws:s3:::$B/*"],"Condition":{"Bool":{"aws:SecureTransport":"false"}}}]}
EOF
aws s3api put-bucket-policy --bucket $B --policy file:///tmp/cap-policy.json

# 3. Versioning + lifecycle (IA at 30d, expire old versions at 90d, abort stale MPUs)
aws s3api put-bucket-versioning --bucket $B --versioning-configuration Status=Enabled
cat > /tmp/cap-lc.json <<'EOF'
{"Rules":[
 {"ID":"artifacts","Status":"Enabled","Filter":{"Prefix":"artifacts/"},
  "Transitions":[{"Days":30,"StorageClass":"STANDARD_IA"}]},
 {"ID":"old-versions","Status":"Enabled","Filter":{},"NoncurrentVersionExpiration":{"NoncurrentDays":90}},
 {"ID":"abort-mpu","Status":"Enabled","Filter":{},"AbortIncompleteMultipartUpload":{"DaysAfterInitiation":7}},
 {"ID":"markers","Status":"Enabled","Filter":{},"Expiration":{"ExpiredObjectDeleteMarker":true}}]}
EOF
aws s3api put-bucket-lifecycle-configuration --bucket $B --lifecycle-configuration file:///tmp/cap-lc.json

# 4. Tags for cost allocation
aws s3api put-bucket-tagging --bucket $B --tagging '{"TagSet":[{"Key":"Project","Value":"s3lab-capstone"}]}'

# 5. Publish a build artifact, with integrity checksum and tags
echo "build-$(date +%s)" > /tmp/app.zip
aws s3api put-object --bucket $B --key artifacts/app.zip --body /tmp/app.zip \
  --checksum-algorithm SHA256 --tagging "env=prod&release=1"

# 6. Share it for 15 minutes without opening the bucket
aws s3 presign s3://$B/artifacts/app.zip --expires-in 900

# 7. Verify the posture
aws s3api get-bucket-policy-status --bucket $B --query PolicyStatus.IsPublic
aws s3api get-bucket-versioning --bucket $B
aws s3api get-bucket-encryption --bucket $B --query 'ServerSideEncryptionConfiguration.Rules[0]'
aws s3api head-object --bucket $B --key artifacts/app.zip --checksum-mode ENABLED \
  --query '[ServerSideEncryption,ChecksumSHA256,VersionId]'

echo "Capstone ready: s3://$B"
```

Tear down (versioned bucket → use `empty_bucket` from Section 29):

```bash
empty_bucket s3lab-${ACCOUNT_ID}-capstone && aws s3 rb s3://s3lab-${ACCOUNT_ID}-capstone
```

---

### Appendix: Concept map

```
aws s3  (high level)           aws s3api (low level)                    aws s3control
  mb rb ls cp mv rm              buckets: create/delete/head/list         access points
  sync presign website           config: versioning, policy, ACL, CORS,   batch operations
                                         lifecycle, encryption, logging,  Storage Lens
                                         replication, notification,       MRAP / Object Lambda
                                         inventory, metrics, tagging,     account-level BPA
                                         accelerate, request-payment,
                                         object-lock, website
                                 objects: put/get/head/copy/delete, MPU,
                                          restore, tagging, ACL, attributes
```

Happy learning, and always remember: **clean up your lab buckets** (Section 29).
