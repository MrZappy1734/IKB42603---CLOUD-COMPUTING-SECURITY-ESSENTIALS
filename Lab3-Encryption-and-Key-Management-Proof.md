# Lab 3: Encryption and Key Management Report

## Course Information

**Course:** IKB42603 Cloud Computing Security Essentials  
**Lab:** Lab 3 - Encryption and Key Management  
**Session:** Sessions A & B (Weeks 5–6)  
**Name:** Hafizi Hasdi  
**Evidence date:** 24 August 2026

## Objective

The objective of this lab is to demonstrate encryption, digital signatures, TLS protection, cloud key management, envelope encryption, key separation, cryptographic erasure, and tamper-evident logging. The activities were completed in a Kali Linux environment using OpenSSL, Docker, Nginx, AWS CLI, and LocalStack KMS.

The report explains the commands used, the security purpose of each activity, and the result shown in the dated evidence screenshots.

## Environment Summary

The local lab used the following components. All screenshots referenced in this report were captured on 24 August 2026.

| Component | Purpose | Evidence |
| --- | --- | --- |
| Kali Linux | Local security testing environment | Screenshots in this report |
| OpenSSL | Symmetric encryption, RSA encryption, signatures, and certificates | Screenshots 1–3 |
| Docker | Run the TLS-enabled Nginx container | Screenshot 3 |
| AWS CLI | Send KMS commands to LocalStack | Screenshots 4–8 |
| LocalStack KMS | Simulate cloud key-management operations locally | Screenshots 4–8 |
| SHA-256 | Detect changes in an audit-log chain | Screenshot 9 |

## Session A (Week 5) — Encryption Fundamentals

## Task 1 — Symmetric Encryption (Data at Rest)

Symmetric encryption uses the same secret key to encrypt and decrypt data. It is efficient for larger data because it does not require a separate public/private key operation for every byte.

### 1.1 Create the confidential record

```bash
echo 'Patient: Ahmad, Diagnosis: confidential' > record.txt
```

The file represents sensitive information that must be protected before storage or transfer.

### 1.2 Encrypt the record with AES-256-CBC

```bash
openssl enc -aes-256-cbc -pbkdf2 -salt \
  -in record.txt -out record.enc
```

OpenSSL prompts for an encryption password and derives an AES key from that password using PBKDF2. The salt makes identical passwords produce different derived encryption values. The resulting `record.enc` file is not readable as the original plaintext.

### 1.3 Decrypt and compare the record

```bash
openssl enc -d -aes-256-cbc -pbkdf2 \
  -in record.enc -out record.dec.txt

diff record.txt record.dec.txt && echo 'MATCH: decryption successful'
```

The comparison returned `MATCH: decryption successful`, proving that the encrypted record could be restored with the correct password.

![AES encryption and successful decryption](Evidence/Screenshot%202026-08-24%20002603.png)

## Task 2 — Asymmetric Encryption & Digital Signatures

Asymmetric cryptography uses two related keys. The public key can be shared, while the private key must remain protected. RSA is useful for key exchange, small messages, and signatures, while symmetric encryption is normally preferred for bulk data.

### 2.1 Generate an RSA key pair

```bash
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

The private key is retained by the owner, and the public key is distributed to parties that need to encrypt data for the owner or verify the owner's signatures.

### 2.2 Encrypt and decrypt the record with RSA

```bash
openssl pkeyutl -encrypt -pubin -inkey public.pem \
  -in record.txt -out record.rsa

openssl pkeyutl -decrypt -inkey private.pem \
  -in record.rsa -out record.rsa.txt
```

The public key encrypted the record, and only the matching private key could decrypt it.

### 2.3 Sign and verify the record

```bash
openssl dgst -sha256 -sign private.pem \
  -out record.sig record.txt

openssl dgst -sha256 -verify public.pem \
  -signature record.sig record.txt
```

The output `Verified OK` confirms that the signature was created with the private key and successfully verified with the public key. This provides integrity and authenticity for the signed file.

![RSA encryption and signature verification](Evidence/Screenshot%202026-08-24%20002629.png)

## Task 3 — Encryption in Transit (TLS)

Encryption at rest is not sufficient if data is exposed while travelling across a network. A self-signed certificate and private key were used to configure an Nginx container to serve the record over HTTPS.

### 3.1 Configure the TLS-enabled Nginx server

The self-signed certificate and private key were generated for the local HTTPS demonstration:

```bash
openssl req -x509 -newkey rsa:2048 \
  -keyout key.pem -out cert.pem \
  -days 7 -nodes -subj '/CN=localhost'
```

The seven-day certificate is suitable for this short-lived local test. It is not a replacement for a certificate issued by a trusted certificate authority in production.

The Nginx configuration used the following security-relevant settings:

```nginx
server {
    listen 443 ssl;
    server_name localhost;
    ssl_certificate /etc/nginx/cert.pem;
    ssl_certificate_key /etc/nginx/key.pem;
    root /usr/share/nginx/html;
}
```

### 3.2 Run the container and test HTTPS access

```bash
docker run --rm -d --name tls -p 8443:443 \
  -v $(pwd)/cert.pem:/etc/nginx/cert.pem \
  -v $(pwd)/key.pem:/etc/nginx/key.pem \
  -v $(pwd)/record.txt:/usr/share/nginx/html/record.txt \
  -v $(pwd)/nginx-tls.conf:/etc/nginx/conf.d/default.conf \
  nginx

sleep 2
curl -k https://localhost:8443/record.txt
```

The Nginx container started successfully and `curl` retrieved the record over the TLS endpoint. The `-k` option is appropriate for this local self-signed certificate because it skips public certificate-authority validation during the demonstration.

![HTTPS Nginx deployment and protected record](Evidence/Screenshot%202026-08-24%20002727.png)

## Session B (Week 6) — Key Management, Envelope Encryption & Erasure

## Task 4 — Create and Use a KMS Master Key

Cloud key-management services protect master keys and provide controlled cryptographic operations. LocalStack KMS was used so that the activity remained local and did not use a real AWS account.

### 4.1 Set the LocalStack endpoint

```bash
EP='--endpoint-url=http://localhost:4566'
```

The endpoint variable ensures that subsequent AWS CLI commands are sent to LocalStack.

### 4.2 Create the tenant-A master key

```bash
aws $EP kms create-key --description 'CCSE tenant-A master key'
```

The response showed an enabled symmetric KMS key with encryption and decryption usage. The returned key ID was stored in `KEY_A` for the following operations.

![Tenant-A KMS master key creation](Evidence/Screenshot%202026-08-24%20002952.png)

## Task 5 — Envelope Encryption

Envelope encryption protects data with a short-lived data key while a KMS master key protects, or wraps, that data key. The plaintext data key is used locally and then removed; the wrapped data key can be retained with the ciphertext.

### 5.1 Encrypt data with the KMS key

```bash
aws $EP kms encrypt --key-id $KEY_A \
  --plaintext "$(echo -n 'hello' | base64)" \
  --query CiphertextBlob --output text
```

This operation returned a KMS ciphertext blob, showing that KMS could encrypt data using the tenant-A master key.

![KMS encryption using tenant-A key](Evidence/Screenshot%202026-08-24%20003502.png)

### 5.2 Generate a data key

```bash
aws $EP kms generate-data-key --key-id $KEY_A \
  --key-spec AES_256 \
  --query '[Plaintext,CiphertextBlob]' --output text
```

KMS returned a plaintext AES-256 data key for local encryption and a ciphertext version of that same data key wrapped by the master key.

![KMS data-key generation](Evidence/Screenshot%202026-08-24%20003619.png)

### 5.3 Encrypt the record locally and retain only the wrapped key

```bash
echo -n '<PLAINTEXT_DATA_KEY_BASE64>' > datakey.b64
echo -n '<WRAPPED_DATA_KEY_BASE64>' > datakey.enc
base64 -d datakey.b64 > datakey.bin

openssl enc -aes-256-cbc -pbkdf2 \
  -in record.txt -out record.env.enc \
  -pass file:./datakey.bin

rm datakey.bin datakey.b64
echo 'Only the KMS-wrapped data key (datakey.enc) remains.'
```

The evidence confirms that the encrypted record and wrapped data key remained after the plaintext data-key material was removed. This reduces exposure because the data key is not stored in plaintext.

![Envelope encryption and removal of plaintext data key](Evidence/Screenshot%202026-08-24%20003800.png)

## Task 6 — Per-Tenant Keys & Cryptographic Erasure

Each tenant should have a separate master key so that a key-management mistake or compromise does not automatically expose every tenant's data.

### 6.1 Create the tenant-B master key

```bash
aws $EP kms create-key --description 'CCSE tenant-B master key'
KEY_B=<PASTE_KEYID>
```

The response showed a separate enabled KMS key for tenant B, with a different key ID from tenant A.

### 6.2 Schedule deletion and test the disabled key

```bash
aws $EP kms schedule-key-deletion \
  --key-id $KEY_A --pending-window-in-days 7

aws $EP kms disable-key --key-id $KEY_A
aws $EP kms decrypt --ciphertext-blob fileb://datakey.enc
```

The KMS response showed the tenant-A key in `PendingDeletion` state with a seven-day pending window. The subsequent decrypt attempt failed because the key was no longer usable. This is the expected security result: destroying or disabling the key makes the wrapped data key unusable, which demonstrates cryptographic erasure of the protected data.

![Tenant-B key separation and tenant-A key deletion result](Evidence/Screenshot%202026-08-24%20003848.png)

## Task 7 — Integrity & Tamper-Evidence

Hash chaining makes changes to earlier audit entries detectable. Each entry includes or depends on the hash of the preceding entry. If an entry is modified, its hash changes and the chain no longer matches the recorded values that follow it.

### 7.1 Compare the original and modified files

```bash
sha256sum record.txt
cp record.txt tampered.txt
echo 'X' >> tampered.txt
sha256sum record.txt tampered.txt
```

The original and modified files produced different SHA-256 values, proving that the appended change was detectable.

### 7.2 Calculate chained hashes for log entries

```bash
PREV=0
for line in 'login ok' 'file read' 'export data'; do
  PREV=$(printf '%s' "$PREV$line" | sha256sum | cut -d' ' -f1)
  echo "$line | $PREV"
done
```

The evidence shows distinct hash values for the audit entries. Each new hash includes the previous hash, so tampering with an earlier entry would break the chain from that point onward.

![SHA-256 tamper detection and audit-log hashes](Evidence/Screenshot%202026-08-24%20003914.png)

## Cleanup

The temporary files and services were removed after the demonstrations:

```bash
docker stop tls 2>/dev/null
rm -f record.* private.pem public.pem key.pem cert.pem datakey.* tampered.txt
docker stop localstack && docker rm localstack
```

The cleanup evidence shows the TLS and LocalStack containers being stopped. Removing temporary keys and encrypted test files reduces the chance of leaving sensitive material in the working directory.

![Lab 3 cleanup](Evidence/Screenshot%202026-08-24%20003946.png)

## Short-Answer Questions

### Q1. Compare symmetric and asymmetric encryption: speed, key distribution, and typical use.

**Answer:** symmetric is faster, but asymmetric is slower. Symmetric use 1 key for bot encrypt, and decrypt, so both sides need the same keys, meanwhile asymmetric use 2 keys for 2 separate purpose. symmetric is usually used for big data(because of it's speed), asymmetric is used for key exchange, digital signature, or small data.

### Q2. Why is key management described as the weakest link, not the algorithm?

**Answer:** it is because of human error, e.g., hardcoded in code, loose access permission, etc. But the algorithm/math itself is impossible to break.

### Q3. Explain envelope encryption and why only the master key needs hardware-grade protection.

**Answer:** master key encrypt the data key, then use it to encrypt the data. the reason why master key needs hardware-grade protection is because it's the single point of maximum impact. Meaning, if the master key is compromised, every data key it ever wrapped is compromised too, which lead to all the data across the whole system is exposed at once.

### Q4. How does cryptographic erasure achieve provable deletion where overwriting cannot (in the cloud)?

**Answer:** it make the data unreadable because of the data encryption, and the erasure of the key to decrypt it. Thus the data can never be used because of the data itself is unusable because of the un-present of the master key. While overwriting can simply left a recoverable fragmented-overwritten data.

### Q5. How does a hash chain make a log tamper-evident (link to tamper-proof logs, Week 6)?

**Answer:** hash value are a unique string of character, and numbers that is unique to a specific file structure. Changing the file meaning changing the structure of the file and changing the hash value. That being said, if someone tampers with an entry from earlier in the log, that entry's hash changes, which breaks the hash of the next entry (since it depended on the old hash), which breaks the one after that, and so on down the whole chain

## Verification Commands

```bash
openssl dgst -sha256 -verify public.pem \
  -signature record.sig record.txt
aws $EP kms list-keys
sha256sum record.txt tampered.txt
docker ps
```

## Environment Verification Checklist

| Check | Status |
| --- | --- |
| AES-256 record encryption completed | Completed |
| AES ciphertext decrypted and matched the original | Completed |
| RSA key pair generated | Completed |
| RSA signature verified with `Verified OK` | Completed |
| HTTPS Nginx container served the record | Completed |
| Tenant-A KMS master key created | Completed |
| Envelope encryption produced a wrapped data key | Completed |
| Plaintext data key material removed | Completed |
| Tenant-B received a separate KMS master key | Completed |
| Tenant-A key deletion/disable behavior demonstrated | Completed |
| Hash changes detected after file tampering | Completed |
| Temporary containers and files cleaned up | Completed |

## Conclusion

Lab 3 was completed successfully on 24 August 2026. The evidence demonstrates encryption at rest with AES, asymmetric encryption and signature verification with RSA, encryption in transit with HTTPS, and local cloud-style key management through LocalStack KMS. The envelope-encryption activity showed how a master key can protect a data key, while separate tenant keys limited the scope of a key-management failure. Scheduling deletion and disabling a key demonstrated cryptographic erasure, and SHA-256 comparisons showed how tampering can be detected in protected audit data.
