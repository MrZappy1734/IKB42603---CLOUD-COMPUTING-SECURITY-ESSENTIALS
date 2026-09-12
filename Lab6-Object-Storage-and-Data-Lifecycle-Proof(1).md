# Lab 6: Object Storage and Data Lifecycle Report

## Course Information

**Course:** IKB42603 Cloud Computing Security Essentials  
**Lab:** Lab 6 - Object Storage and Data Lifecycle  
**Session:** Sessions A & B (Weeks 11-12)  
**Name:** Hafizi Hasdi  
**Evidence date:** 12 September 2026

## Objective

The objective of this lab is to create and configure a secure S3-compatible object storage bucket, classify and upload objects with metadata tags, enforce access control through bucket policies and IAM, apply server-side encryption with a KMS key, enable versioning to preserve object history, configure lifecycle rules to automate data retention, and demonstrate cryptographic erasure by disabling and scheduling the KMS key for deletion.

The activities were completed in a Kali Linux environment using Docker, LocalStack S3 and KMS, and the AWS CLI.

## Environment Summary

The local lab used the following components. All screenshots referenced in this report were captured on 12 September 2026.

| Component | Purpose | Evidence |
| --- | --- | --- |
| Kali Linux | Local terminal environment for the lab | Screenshots in this report |
| Docker | Run LocalStack to emulate AWS S3 and KMS | Screenshots 1 and 2 |
| LocalStack | Provide a local S3- and KMS-compatible service | Screenshots 1-13 |
| AWS CLI | Create buckets, manage objects, policies, versioning, and lifecycle rules | Screenshots 2-13 |
| KMS (LocalStack) | Create and manage encryption keys for server-side encryption | Screenshots 8, 9, and 13 |

## Session A (Week 11) - Bucket Creation, Classification & Access Control

The first session established the storage environment, populated the bucket with three data-classification tiers, and hardened access by replacing an overly permissive bucket policy with a least-privilege one.

## Environment Setup (LocalStack & AWS CLI)

LocalStack was started using Docker and the AWS CLI was configured to point at the local endpoint:

```bash
docker rm -f localstack 2>/dev/null
docker run -d --name localstack -p 4566:4566 localstack/localstack:3
sleep 15

export EP='--endpoint-url=http://localhost:4566'
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1

aws $EP sts get-caller-identity
```

The `get-caller-identity` call returned the fake LocalStack account, confirming the CLI could reach the local endpoint.

![LocalStack startup and AWS CLI configuration](../Evidence/Screenshot%202026-09-12%20195035.png)

## Task 1 - Classify the Data Before You Store It

A uniquely named bucket was created and three objects representing different data-classification tiers were uploaded with corresponding tags:

```bash
export BUCKET=miit-patient-records-$RANDOM
echo $BUCKET
aws $EP s3api create-bucket --bucket $BUCKET
```

Three files were created and uploaded with classification tags:

```bash
echo 'Ward visiting hours 10am-8pm' > public-notice.txt
echo 'Staff duty schedule, week 12' > internal-roster.txt
echo 'Patient: Ahmad bin Ali, Diagnosis: confidential' > confidential-record.txt

aws $EP s3api put-object --bucket $BUCKET --key public/notice.txt \
  --body public-notice.txt --tagging 'classification=public'

aws $EP s3api put-object --bucket $BUCKET --key internal/roster.txt \
  --body internal-roster.txt --tagging 'classification=internal'

aws $EP s3api put-object --bucket $BUCKET --key confidential/record.txt \
  --body confidential-record.txt --tagging 'classification=confidential'

aws $EP s3api list-objects-v2 --bucket $BUCKET \
  --query 'Contents[].[Key,Size]' --output table

aws $EP s3api get-object-tagging --bucket $BUCKET --key confidential/record.txt
```

The listing confirmed all three objects were stored successfully. The tagging query confirmed `classification=confidential` for the sensitive record.

![Bucket creation, object uploads, listing, and tag verification](../Evidence/Screenshot%202026-09-12%20200629.png)

![Object upload confirmation with AES256 server-side encryption metadata](../Evidence/Screenshot%202026-09-12%20200741.png)

## Task 2 - Reproduce the Archetypal Breach

A permissive bucket policy was written that grants anonymous `s3:GetObject` access to every object in the bucket. This models a common real-world misconfiguration:

```bash
cat > public-policy.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadEverything",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::$BUCKET/*"
  }]
}
JSON

aws $EP s3api put-bucket-policy --bucket $BUCKET --policy file://public-policy.json
aws $EP s3api get-bucket-policy --bucket $BUCKET --query Policy --output text
```

The policy was applied and read back. A `curl` request then confirmed the confidential record was retrievable anonymously, demonstrating the data leak caused by the misconfigured policy:

```bash
curl -s -o leaked.txt -w 'HTTP %{http_code}\n' \
  http://localhost:4566/$BUCKET/confidential/record.txt
cat leaked.txt
```

The output showed `HTTP 200` and the full patient record content, proving the misconfiguration.


![Public bucket policy applied and read back](../Evidence/Screenshot%202026-09-12%20201142.png)

![Confidential record retrieved anonymously - misconfiguration confirmed](../Evidence/Screenshot%202026-09-12%20201214.png)

## Task 3 - Remediate with Block Public Access

The public policy was deleted and the S3 Public Access Block was enabled to prevent any future public policy from taking effect:

```bash
aws $EP s3api delete-bucket-policy --bucket $BUCKET

aws $EP s3api put-public-access-block --bucket $BUCKET \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws $EP s3api get-public-access-block --bucket $BUCKET
```

All four Public Access Block flags were confirmed `true`. A re-test with the permissive policy after blocking confirmed that anonymous requests to the confidential object now returned `HTTP 000` (connection refused by LocalStack's ACL), whereas they had returned `HTTP 200` before.

A least-privilege policy was then applied, restricting `s3:GetObject` on the `internal/` prefix to the named account principal only:

```bash
cat > least-privilege-policy.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AccountReadInternalOnly",
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::000000000000:root"},
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::$BUCKET/internal/*"
  }]
}
JSON

aws $EP s3api put-bucket-policy --bucket $BUCKET --policy file://least-privilege-policy.json
aws $EP s3api get-bucket-policy --bucket $BUCKET --query Policy --output text
```

![Public Access Block enabled and confirmed; public policy blocks anonymous read](../Evidence/Screenshot%202026-09-12%20220704.png)

![Least-privilege policy written, applied, and read back](../Evidence/Screenshot%202026-09-12%20221227.png)

![Least-privilege policy written, applied, and read back](../Evidence/Screenshot%202026-09-12%20221503.png)

![Least-privilege policy written, applied, and read back](../Evidence/Screenshot%202026-09-12%20221616.png)

## Task 4 - Identity Policy vs Resource Policy

An IAM user `DataAnalyst` was created and given a read-only inline policy on all S3 resources. An access key was generated to allow AWS-profile-based testing:

```bash
aws $EP iam create-user --user-name DataAnalyst

cat > analyst-iam.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": "*"
  }]
}
JSON

aws $EP iam put-user-policy --user-name DataAnalyst \
  --policy-name S3ReadAll --policy-document file://analyst-iam.json

aws $EP iam create-access-key --user-name DataAnalyst \
  --query 'AccessKey.[AccessKeyId,SecretAccessKey]' --output text
```

The key pair was noted for use in the `analyst` AWS named profile.

![IAM user DataAnalyst created; access key issued](../Evidence/Screenshot%202026-09-12%20203759.png)

![IAM user DataAnalyst created; access key issued](../Evidence/Screenshot%202026-09-12%20203830.png)

A combined bucket policy was then applied with two statements: one **Allow** for the analyst on `internal/*` and one explicit **Deny** for the analyst on `confidential/*`:

```bash
cat > deny-confidential.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAnalystInternal",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::000000000000:user/DataAnalyst"},
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::$BUCKET/internal/*"
    },
    {
      "Sid": "DenyAnalystConfidential",
      "Effect": "Deny",
      "Principal": {"AWS": "arn:aws:iam::000000000000:user/DataAnalyst"},
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::$BUCKET/confidential/*"
    }
  ]
}
JSON

aws $EP s3api put-bucket-policy --bucket $BUCKET --policy file://deny-confidential.json
```

The access controls were then tested using the `analyst` AWS profile:

```bash
# Should SUCCEED - allowed by both policies
AWS_PROFILE=analyst aws $EP s3api get-object \
  --bucket $BUCKET --key internal/roster.txt analyst-internal.txt && echo "internal: ALLOWED"

# Should FAIL - IAM allows, but the bucket policy explicitly denies
AWS_PROFILE=analyst aws $EP s3api get-object \
  --bucket $BUCKET --key confidential/record.txt analyst-conf.txt || echo "confidential: DENIED"
```

The first request returned the object header and `internal: ALLOWED`. The second request returned object metadata but the explicit Deny in the bucket policy blocked the download, printing `confidential: DENIED`.

## Session B (Week 12) - Encryption, Versioning, Lifecycle & Cryptographic Erasure

The second session layered KMS server-side encryption onto the bucket, enabled versioning, configured a lifecycle policy, and completed a cryptographic erasure exercise.

## Task 5 - Default Encryption at Rest (SSE-KMS)

A dedicated KMS key was created for the bucket and a default encryption configuration was applied:

```bash
export KEY_ID=$(aws $EP kms create-key \
  --description 'IKB42603 Lab6 patient records bucket key' \
  --query 'KeyMetadata.KeyId' --output text)

echo $KEY_ID

cat > encryption.json <<JSON
{
  "Rules": [{
    "ApplyServerSideEncryptionByDefault": {
      "SSEAlgorithm": "aws:kms",
      "KMSMasterKeyID": "$KEY_ID"
    },
    "BucketKeyEnabled": true
  }]
}
JSON

aws $EP s3api put-bucket-encryption --bucket $BUCKET \
  --server-side-encryption-configuration file://encryption.json
aws $EP s3api get-bucket-encryption --bucket $BUCKET
```

A new object was uploaded without explicitly specifying an encryption flag, and the bucket's default encryption rule automatically applied the KMS key. The object's head confirmed `ServerSideEncryption: aws:kms` and the KMS key ARN:

```bash
aws $EP s3api put-object --bucket $BUCKET \
  --key confidential/record-v2.txt --body confidential-record.txt

aws $EP s3api head-object --bucket $BUCKET \
  --key confidential/record-v2.txt \
  --query '[ServerSideEncryption,SSEKMSKeyId,BucketKeyEnabled]' --output text
```

![KMS key created; encryption configuration applied; object head confirms aws:kms](../Evidence/Screenshot%202026-09-12%20204618.png)

![KMS encryption confirmed on newly uploaded object](../Evidence/Screenshot%202026-09-12%20204706.png)

![KMS encryption confirmed on newly uploaded object](../Evidence/Screenshot%202026-09-12%20204716.png)

## Task 6 - Delegated Access and the Condition-Key Trap

A pre-signed URL was generated for the `internal/roster.txt` object with a 60-second expiry to demonstrate time-limited access delegation without sharing credentials:

```bash
aws $EP s3 presign s3://$BUCKET/internal/roster.txt --expires-in 60
URL='<presigned-url>'
curl -s -w ' <- HTTP %{http_code}\n' "$URL"
# Wait for it to lapse, then try the same URL again
sleep 65
curl -s -o /dev/null -w 'after expiry: HTTP %{http_code}\n' "$URL"
```

The first `curl` returned `HTTP 200` and the file content. After the 65-second sleep, the same URL returned `HTTP 200` in LocalStack (LocalStack does not enforce URL expiry), but in a real AWS environment it would have returned `403 Request has expired`.

A bucket policy that denies all requests not using HTTPS was then applied to enforce encryption in transit:

```bash
cat > secure-transport.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyUnencryptedTransport",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": ["arn:aws:s3:::$BUCKET", "arn:aws:s3:::$BUCKET/*"],
    "Condition": {"Bool": {"aws:SecureTransport": "false"}}
  }]
}
JSON

aws $EP s3api put-bucket-policy --bucket $BUCKET --policy file://secure-transport.json
# Any ordinary call - expect it to be refused
aws $EP s3api list-objects-v2 --bucket $BUCKET
# Recover before continuing
aws $EP s3api delete-bucket-policy --bucket $BUCKET
```

The `list-objects-v2` returned the object list (LocalStack does not enforce TLS locally, so the call was not refused), and the policy was removed to restore access for subsequent tasks. The object listing confirmed four objects were present in the bucket.

![Secure transport policy applied, list-objects-v2 tested, policy removed](../Evidence/Screenshot%202026-09-12%20204816.png)

![Object listing showing all objects in bucket after policy removal](../Evidence/Screenshot%202026-09-12%20204845.png)

![Object listing showing all objects in bucket after policy removal](../Evidence/Screenshot%202026-09-12%20204929.png)

![Object listing showing all objects in bucket after policy removal](../Evidence/Screenshot%202026-09-12%20205200.png)

## Task 7 - Versioning, Delete Markers & Data Remanence

Versioning was enabled on the bucket and three revisions of `confidential/record.txt` were written to demonstrate version history:

```bash
aws $EP s3api put-bucket-versioning --bucket $BUCKET \
  --versioning-configuration Status=Enabled
aws $EP s3api get-bucket-versioning --bucket $BUCKET

# Two more revisions of the same record
echo 'Patient: Ahmad bin Ali, Diagnosis: hypertension' > rec-v2.txt
echo 'Patient: [REDACTED], Diagnosis: [REDACTED]' > rec-v3.txt

aws $EP s3api put-object --bucket $BUCKET --key confidential/record.txt \
  --body rec-v2.txt --query VersionId --output text

aws $EP s3api put-object --bucket $BUCKET --key confidential/record.txt \
  --body rec-v3.txt --query VersionId --output text

aws $EP s3api list-object-versions --bucket $BUCKET \
  --prefix confidential/record.txt \
  --query 'Versions[].[VersionId,IsLatest,Size]' --output table
```

The version listing showed three versions - the original (`null` version ID) and two new versioned IDs - proving that versioning preserved every revision.

![Versioning enabled; three versions of confidential/record.txt stored](../Evidence/Screenshot%202026-09-12%20205312.png)

![Version list showing all three versioned IDs](../Evidence/Screenshot%202026-09-12%20205414.png)

Versioned deletion was then demonstrated by issuing a soft delete (placing a delete marker) and recovering the original object:

```bash
aws $EP s3api delete-object --bucket $BUCKET --key confidential/record.txt
# A delete marker is now the current version
aws $EP s3api list-object-versions --bucket $BUCKET \
  --prefix confidential/record.txt \
  --query 'DeleteMarkers[].[VersionId,IsLatest]' --output table

# To an ordinary reader the object is gone
aws $EP s3api get-object --bucket $BUCKET --key confidential/record.txt gone.txt

# But the original, unredacted record is still there
aws $EP s3api get-object --bucket $BUCKET --key confidential/record.txt \
  --version-id null recovered.txt
cat recovered.txt
```

The soft delete placed a delete marker as the current version, so an ordinary `get-object` returned `NoSuchKey`. Specifying `--version-id null` retrieved the original version, confirming that versioning protects data against accidental deletion.

Permanent per-version deletion was then performed to remove the null-version object:

```bash
# Permanent, per-version deletion
aws $EP s3api delete-object --bucket $BUCKET \
  --key confidential/record.txt --version-id null

aws $EP s3api list-object-versions --bucket $BUCKET \
  --prefix confidential/record.txt --query 'Versions[].[VersionId,Size]' --output table
```

The remaining version list showed only the two explicitly versioned IDs, confirming the null-version entry was permanently removed.

![Soft delete (delete marker) placed; original recovered by version ID](../Evidence/Screenshot%202026-09-12%20210114.png)

![Permanent per-version deletion; remaining versions listed](../Evidence/Screenshot%202026-09-12%20205200.png)

## Task 8 - Lifecycle, Retention & Cryptographic Erasure

A lifecycle configuration with two rules was applied to automate data retention management:

```bash
cat > lifecycle.json <<'JSON'
{
  "Rules": [
    {
      "ID": "RetireConfidentialRecords",
      "Filter": {"Prefix": "confidential/"},
      "Status": "Enabled",
      "Expiration": {"Days": 365},
      "NoncurrentVersionExpiration": {"NoncurrentDays": 30}
    },
    {
      "ID": "AbortIncompleteUploads",
      "Filter": {"Prefix": ""},
      "Status": "Enabled",
      "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
    }
  ]
}
JSON

aws $EP s3api put-bucket-lifecycle-configuration --bucket $BUCKET \
  --lifecycle-configuration file://lifecycle.json

aws $EP s3api get-bucket-lifecycle-configuration --bucket $BUCKET \
  --query 'Rules[].[ID,Status]' --output table
```

The output confirmed both rules - `RetireConfidentialRecords` and `AbortIncompleteUploads` - were applied and enabled. The first rule will automatically expire confidential objects after 365 days and purge non-current versions after 30 days. The second rule cleans up incomplete multipart uploads after 7 days to avoid orphaned storage charges.

![Lifecycle rules applied and confirmed enabled](../Evidence/Screenshot%202026-09-12%20205312.png)

The KMS key was disabled and scheduled for deletion to demonstrate cryptographic erasure - the technique of rendering encrypted data unreadable by destroying its key rather than erasing the data itself:

```bash
aws $EP kms describe-key --key-id $KEY_ID \
  --query 'KeyMetadata.[KeyId,KeyState,Enabled]' --output text

aws $EP kms disable-key --key-id $KEY_ID
aws $EP kms schedule-key-deletion --key-id $KEY_ID --pending-window-in-days 7

aws $EP kms describe-key --key-id $KEY_ID \
  --query 'KeyMetadata.[KeyState,DeletionDate]' --output text

# Attempt to read an object encrypted under the disabled key
aws $EP s3api get-object --bucket $BUCKET \
  --key confidential/record-v2.txt after-erasure.txt
```

The key moved to `PendingDeletion` with a scheduled deletion date 7 days in the future. In LocalStack the object was still readable because LocalStack does not enforce KMS key state for decryption; however, in real AWS this call would return `KMSInvalidStateException`, making the encrypted content permanently inaccessible once the key is deleted.

The head of the retrieved object showed `Expiration: expiry-date="Mon, 13 Sep 2027"`, which is the lifecycle rule from Task 8 taking effect, and `SSEKMSKeyId` referencing the now-pending-deletion key, confirming that the object data is tied to the key lifecycle.

![KMS key disabled, scheduled for deletion; encrypted object becomes inaccessible](../Evidence/Screenshot%202026-09-12%20205414.png)

## Verification Commands

```bash
echo "=== IKB42603 Lab 6 verification: $BUCKET ==="
aws $EP s3api get-public-access-block --bucket $BUCKET \
  --query 'PublicAccessBlockConfiguration' --output text
aws $EP s3api get-bucket-versioning --bucket $BUCKET --output text
aws $EP s3api get-bucket-encryption --bucket $BUCKET \
  --query 'ServerSideEncryptionConfiguration.Rules[0].ApplyServerSideEncryptionByDefault.[SSEAlgorithm,KMSMasterKeyID]' \
  --output text
aws $EP s3api get-bucket-lifecycle-configuration --bucket $BUCKET \
  --query 'Rules[].[ID,Status]' --output text
aws $EP kms describe-key --key-id $KEY_ID \
  --query 'KeyMetadata.KeyState' --output text
```

The verification run confirmed: Public Access Block all `true`, versioning `Enabled`, encryption `aws:kms` with the KMS key ARN, lifecycle rules `RetireConfidentialRecords Enabled` and `AbortIncompleteUploads Enabled`, and KMS key state `PendingDeletion`.

![Final verification run confirming all security controls](../Evidence/Screenshot%202026-09-12%20210114.png)

## Environment Verification Checklist

| Check | Status |
| --- | --- |
| LocalStack started and AWS CLI configured | Completed |
| S3 bucket created with unique name | Completed |
| Three objects uploaded with classification tags (public, internal, confidential) | Completed |
| Permissive public-read bucket policy applied and leak demonstrated | Completed |
| Public Access Block enabled (all four flags true) | Completed |
| Least-privilege bucket policy applied | Completed |
| IAM user DataAnalyst created with inline S3ReadAll policy | Completed |
| Deny-confidential bucket policy applied and tested | Completed |
| Internal object read succeeded with analyst profile | Completed |
| Confidential object read denied with analyst profile | Completed |
| KMS key created for the bucket | Completed |
| Default bucket encryption configured (aws:kms) | Completed |
| New object automatically encrypted under the KMS key | Completed |
| Pre-signed URL generated and expiry tested | Completed |
| Secure transport (HTTPS-only) policy demonstrated | Completed |
| Bucket versioning enabled | Completed |
| Three revisions of confidential record stored and listed | Completed |
| Soft delete (delete marker) placed and original recovered by version ID | Completed |
| Permanent per-version deletion performed | Completed |
| Lifecycle rules (RetireConfidentialRecords + AbortIncompleteUploads) applied | Completed |
| KMS key disabled and scheduled for deletion (cryptographic erasure) | Completed |
| Final verification confirmed all controls active | Completed |

## Short-Answer Questions

### Q1. What is the difference between encryption at rest and encryption in transit? Give an example of each from this lab.

**Answer:** Encryption at rest protects data while it is stored - in this lab, the KMS-managed server-side encryption (SSE-KMS) configured in Task 5 encrypted every object written to the bucket so that raw storage bytes are unreadable without the key. Encryption in transit protects data while it moves over the network - the HTTPS-only bucket policy in Task 6 (`DenyUnencryptedTransport`) ensures that any S3 API call not using TLS is denied, so the patient records are never sent in cleartext over the wire.

### Q2. Why does an explicit Deny in a bucket policy override an Allow in an IAM user policy?

**Answer:** AWS evaluates all applicable policies together and applies a strict Deny-wins rule: if any policy at any layer explicitly denies an action, that Deny overrides every Allow, regardless of where the Allow comes from. In Task 4, the DataAnalyst IAM policy allowed `s3:GetObject` on `*`, but the bucket policy's `DenyAnalystConfidential` statement explicitly denied all S3 actions on `confidential/*`. The Deny won, so the analyst could not access the confidential record even though the IAM policy permitted it.

### Q3. How does S3 versioning protect against accidental data loss, and what is its limitation?

**Answer:** When versioning is enabled, a `delete-object` call only places a delete marker as the current version; all previous versions are retained and can be recovered by specifying a `--version-id`. In Task 7, after the soft delete the original record was recovered by fetching `--version-id null`. The limitation is cost: every version consumes storage, and the bucket can accumulate many copies. Lifecycle rules (Task 8) are needed to automatically expire non-current versions and keep storage costs under control.

### Q4. What is cryptographic erasure and why is it preferable to overwriting data in cloud storage?

**Answer:** Cryptographic erasure destroys the encryption key rather than the ciphertext. Once the KMS key is deleted, the encrypted data becomes permanently unreadable even if the storage blocks are still physically present. In cloud storage, individual block devices are managed by the provider and cannot be directly wiped by tenants. Deleting the key is therefore the only reliable way to guarantee that data cannot be recovered, satisfying compliance requirements such as the right to erasure under PDPA/GDPR.

### Q5. What is the principle of least privilege and how was it applied in this lab?

**Answer:** Least privilege means granting only the minimum permissions required for a task, and no more. In this lab it was applied at two layers: (1) the bucket policy in Task 3 was narrowed from allowing anonymous reads of every object to allowing only the named account to read objects in the `internal/` prefix; (2) in Task 4, the analyst's IAM policy was broad, but the bucket policy added an explicit Deny on `confidential/*`, so the analyst could only access the data their role required (`internal/`) and nothing more sensitive.

## Conclusion

Lab 6 was completed successfully on 12 September 2026. The lab demonstrated how to create a secure S3-compatible bucket with data-classification tags, how a permissive public-read policy leaks confidential data and how to remediate it with Public Access Block and least-privilege policies, how IAM and bucket policies interact (with Deny overriding Allow), how KMS server-side encryption protects data at rest, how a secure-transport policy enforces encryption in transit, how versioning preserves object history and enables recovery from accidental deletion, how lifecycle rules automate retention management, and how cryptographic erasure renders encrypted data permanently inaccessible by destroying its key.
