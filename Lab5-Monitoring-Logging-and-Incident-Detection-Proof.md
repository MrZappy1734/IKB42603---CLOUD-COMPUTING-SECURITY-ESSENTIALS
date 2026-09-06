# Lab 5: Monitoring, Logging and Incident Detection Report

## Course Information

**Course:** IKB42603 Cloud Computing Security Essentials  
**Lab:** Lab 5 - Monitoring, Logging & Incident Detection  
**Session:** Sessions A & B (Weeks 9–10)  
**Name:** Hafizi Hasdi  
**Evidence date:** 6 September 2026

## Objective

The objective of this lab is to collect and centralise application logs, query logs for security-relevant activity, build a tamper-evident hash-chained log, detect an incident by correlating multiple events, and perform the incident-response steps of containment, evidence collection, and documentation.

The activities were completed in a Kali Linux environment using Docker, LocalStack CloudWatch Logs, AWS CLI, `grep`, `awk`, `sha256sum`, and standard shell commands.

## Environment Summary

The local lab used the following components. All screenshots referenced in this report were captured on 6 September 2026.

| Component | Purpose | Evidence |
| --- | --- | --- |
| Kali Linux | Local terminal environment for the lab | Screenshots in this report |
| Docker | Run LocalStack and model the containment rule | Screenshots 1, 7, and 9 |
| LocalStack | Provide a local CloudWatch Logs-compatible service | Screenshots 1, 3, and 8 |
| AWS CLI | Create log resources, ship logs, read them back, and verify log groups | Screenshots 1, 3, and 8 |
| `grep` and `awk` | Query failed-login activity by IP address | Screenshot 4 |
| `sha256sum` | Build and verify the tamper-evident log chain and evidence copy | Screenshots 5, 7, and 8 |

## Session A (Week 9) — Logging & Centralisation Setup

The first session generated application authentication events and centralised them in a LocalStack CloudWatch Logs stream. The centralised copy can be queried later for monitoring and incident investigation.

## Task 1 — Generate Application Logs

The following log contains a normal login, repeated failed logins from the same external IP address, a successful login, and a large data export:

```bash
cat > auth.log <<'EOF'
2025-03-01T09:00:01 LOGIN_OK    user=ahmad   ip=10.0.0.5
2025-03-01T09:01:10 LOGIN_FAIL  user=admin   ip=203.0.113.9
2025-03-01T09:01:12 LOGIN_FAIL  user=admin   ip=203.0.113.9
2025-03-01T09:01:15 LOGIN_FAIL  user=admin   ip=203.0.113.9
2025-03-01T09:01:18 LOGIN_FAIL  user=admin   ip=203.0.113.9
2025-03-01T09:01:22 LOGIN_OK    user=admin   ip=203.0.113.9
2025-03-01T09:01:40 EXPORT_DATA user=admin   ip=203.0.113.9 size=500MB
EOF

cat auth.log
```

The generated log records the sequence that will later be detected as a probable brute-force attack followed by account compromise and data exfiltration.

![Generated authentication log](Evidence/Screenshot%202026-09-06%20122221.png)

## Task 2 — Centralise Logs (Ship to CloudWatch)

LocalStack was started and configured as the local CloudWatch Logs endpoint:

```bash
docker run -d --name localstack -p 4566:4566 localstack/localstack

EP='--endpoint-url=http://localhost:4566'
aws $EP logs create-log-group --log-group-name /ccse/app
aws $EP logs create-log-stream \
  --log-group-name /ccse/app \
  --log-stream-name auth
```

Each line of `auth.log` was sent to the central log stream with an increasing timestamp:

```bash
TS=$(date +%s000)
while IFS= read -r line; do
  aws $EP logs put-log-events \
    --log-group-name /ccse/app \
    --log-stream-name auth \
    --log-events timestamp=$TS,message="$line" >/dev/null
  TS=$((TS+1000))
done < auth.log
```

The centralised log was then read back from LocalStack:

```bash
aws $EP logs get-log-events \
  --log-group-name /ccse/app \
  --log-stream-name auth \
  --query 'events[].message' --output text
```

The read-back output contains the same authentication events as the local file, proving that the application logs were shipped to and retrieved from the central service.

![LocalStack log group and stream setup](Evidence/Screenshot%202026-09-06%20122034.png)

![Centralised log read-back](Evidence/Screenshot%202026-09-06%20140325.png)

## Task 3 — Query for Security-Relevant Activity

The failed-login records were filtered and grouped by IP address:

```bash
grep LOGIN_FAIL auth.log | awk '{print $4, $5}' | sort | uniq -c
```

The output shows four failed logins associated with `203.0.113.9`. This is a log query that identifies suspicious activity. An event would be a near-real-time trigger created from that activity, such as:

```text
ALERT: 4 failures from 203.0.113.9
```

The log is the durable record; the event is the alert or trigger generated from one or more records.

![Failed-login count grouped by IP](Evidence/Screenshot%202026-09-06%20140354.png)

## Session B (Week 10) — Tamper-Proofing, Detection & Response

The second session protected the log history against undetected modification, correlated multiple events into an incident detection, and performed containment and evidence collection.

## Task 4 — Tamper-Proof (Hash-Chained) Logs

Each line was chained to the hash of the preceding line. The resulting chain was saved as `auth.chain`:

```bash
PREV=0
while IFS= read -r line; do
  PREV=$(printf '%s%s' "$PREV" "$line" | sha256sum | cut -d' ' -f1)
  printf '%s | %s\n' "$line" "$PREV"
done < auth.log > auth.chain

cat auth.chain
```

The chain was then tested against a modified copy of the log. The export size was changed from `500MB` to `5MB` and the chain was recomputed:

```bash
sed 's/500MB/5MB/' auth.log > auth.tampered

PREV=0
while IFS= read -r line; do
  PREV=$(printf '%s%s' "$PREV" "$line" | sha256sum | cut -d' ' -f1)
done < auth.tampered

echo "Original final hash:"
tail -1 auth.chain | awk -F'|' '{print $2}'
echo "Tampered final hash:"
echo "$PREV"
```

Changing one log entry changed the final hash. The different final hash proves that the tampered log is not identical to the original hash chain. In a production design, the final hash or the chain would also be forwarded to a separate append-only location so that an attacker who can edit the application log cannot silently rewrite its audit trail.

![Hash chain and tampered-log comparison](Evidence/Screenshot%202026-09-06%20140707.png)

## Task 5 — Detect the Incident (Correlation)

The events were correlated for the suspicious IP address:

```bash
IP=203.0.113.9
FAILS=$(grep -c "LOGIN_FAIL.*$IP" auth.log)
SUCCESS=$(grep -c "LOGIN_OK.*$IP" auth.log)
EXPORT=$(grep -c "EXPORT_DATA.*$IP" auth.log)

echo "IP=$IP fails=$FAILS success=$SUCCESS export=$EXPORT"

if [ "$FAILS" -ge 3 ] && [ "$SUCCESS" -ge 1 ] && [ "$EXPORT" -ge 1 ]; then
  echo 'ALERT: probable brute-force -> compromise -> data exfiltration'
fi
```

The result was:

```text
IP=203.0.113.9 fails=4 success=1 export=1
ALERT: probable brute-force -> compromise -> data exfiltration
```

No single log line proves the complete incident. The correlation rule detects the sequence of repeated failures, a successful login, and a large export from the same IP address. This models the type of multi-event detection performed by a SIEM.

![Incident correlation alert](Evidence/Screenshot%202026-09-06%20140819.png)

## Task 6 — Incident Response

### Contain

The suspicious IP address was blocked using an iptables rule inside a temporary Alpine container:

```bash
docker run --rm --cap-add=NET_ADMIN alpine sh -c \
  'apk add -q iptables; \
   iptables -A INPUT -s 203.0.113.9 -j DROP; \
   iptables -L INPUT -n | tail -2'
```

The output showed a `DROP` rule for source address `203.0.113.9`, modelling containment of the suspected attacker.

![Containment rule and evidence hash creation](Evidence/Screenshot%202026-09-06%20141001.png)

### Collect Evidence & Integrity

An evidence copy was created with the date in its filename and then hashed:

```bash
cp auth.log evidence_$(date +%Y%m%d).log
sha256sum evidence_*.log > evidence.sha256
cat evidence.sha256
```

The hash file records the integrity value for `evidence_20260906.log`, allowing the collected evidence to be checked later.

The verification command confirmed both the central log group and the evidence file hash:

```bash
aws --endpoint-url=http://localhost:4566 logs describe-log-groups
sha256sum -c evidence.sha256
```

The output reported `evidence_20260906.log: OK`, confirming that the evidence file had not changed after its hash was recorded.

![Central log-group and evidence-integrity verification](Evidence/Screenshot%202026-09-06%20141212.png)

## Incident Report

### Detection

Four failed login attempts from `203.0.113.9` were followed by one successful login and one `EXPORT_DATA` event involving `500MB`. The correlation rule generated the alert: `probable brute-force -> compromise -> data exfiltration`.

### Analysis

The sequence indicates that the source IP repeatedly attempted to authenticate as `admin`, eventually succeeded, and then exported a large amount of data. The individual records are not enough to show the full incident, but their order and common source IP reveal the likely attack sequence.

### Containment

The suspected source address `203.0.113.9` was blocked with an iptables `DROP` rule. This prevents further input from the address in the containment model.

### Evidence & integrity

The original `auth.log` was copied to `evidence_20260906.log`. Its SHA-256 digest was stored in `evidence.sha256`, and `sha256sum -c evidence.sha256` returned `OK`. The original log was also centralised in the `/ccse/app` CloudWatch Logs group and protected with a hash chain. Recomputing the chain after changing `500MB` to `5MB` produced a different final hash.

### Lesson learned

Centralised logs, tamper-evident records, and correlation rules are necessary because a single event may appear harmless. Keeping an independently hashed evidence copy helps preserve the integrity of the incident timeline for investigation and compliance.

## Short-Answer Questions

### Q1. What is the difference between a log and an event? Give an example of each from this lab.

**Answer:** a log is a durable, stored record you can go back and query later. An event is a real-time trigger or an alert fired the moment something happens.

### Q2. Why must audit logs be tamper-proof, and how does a hash chain achieve this?

**Answer:** audit logs needs to be tamper-proof because to maintain a CIA triad most importantly Integrity. hash chain achieve by presenting a completely different hash chain if the previous hash chain is altered. The mismatch will be the proof of tempering.

### Q3. How did correlation detect an incident that no single log line revealed?

**Answer:** Correlation connected four failed logins, one successful login, and one large data export from the same IP address. Together, these events showed the pattern of brute force, possible compromise, and data exfiltration.

### Q4. List the incident-response steps you performed and the goal of each.

**Answer:**

1. Detect — correlate the failed logins + success + export to recognize an incident occurred.
2. Contain — block the attacker's IP to stop further damage.
3. Collect evidence — make an immutable, hashed copy of the log for forensics.
4. Document — write the incident report/timeline.

### Q5. How do the same logs serve both security monitoring and compliance evidence (Weeks 6, 11)?

**Answer:** In security monitoring, logs let cybersecurity detect and correlate suspicious activity in near-real-time. While in Compliance evidence, the same logs, once they're tamper-evident and centrally stored, serve as an audit trail proving to regulators/auditors that security controls were actually in place and that incidents were properly detected and responded to.

## Verification Commands

```bash
aws --endpoint-url=http://localhost:4566 logs describe-log-groups
sha256sum -c evidence.sha256
```

## Environment Verification Checklist

| Check | Status |
| --- | --- |
| LocalStack log group and stream created | Completed |
| Authentication log generated | Completed |
| Logs shipped to the central CloudWatch-compatible service | Completed |
| Centralised logs read back successfully | Completed |
| Failed-login activity queried and grouped by IP | Completed |
| Hash-chained log created | Completed |
| Tampered log produced a different final hash | Completed |
| Incident detected through event correlation | Completed |
| Suspected attacker IP contained with a DROP rule | Completed |
| Dated evidence copy created and hashed | Completed |
| Evidence hash verification returned `OK` | Completed |
| Incident report documented | Completed |

## Cleanup & Teardown

The temporary log files and LocalStack container were removed after the evidence was captured:

```bash
rm -f auth.log auth.chain auth.tampered evidence_*.log evidence.sha256
docker stop localstack && docker rm localstack
```

![Lab cleanup](Evidence/Screenshot%202026-09-06%20141236.png)

## Conclusion

Lab 5 was completed successfully on 6 September 2026. The lab demonstrated how centralised logging provides visibility, how queries identify security-relevant activity, how hash chaining detects log alteration, and how correlation turns separate records into an incident alert. The response exercise then contained the suspected source, collected a dated evidence copy, verified its integrity, and documented the incident timeline and lesson learned.
