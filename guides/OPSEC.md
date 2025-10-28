# Operations & Security Manual

Operational procedures and security guidelines for GigVault PKI.

## Root CA Operations

### Initial Root CA Generation

**Environment**: Air-gapped system (never connected to network)

**Procedure**:

1. **Prepare Secure System**
   ```bash
   # Use dedicated, wiped hardware
   # Disconnect all network interfaces
   # Verify isolation
   ip link show  # Should show no active interfaces
   ```

2. **Generate Root CA**
   ```bash
   cd rootca
   export CN="GigVault Root CA"
   export VALIDITY_YEARS=20
   export OUTPUT_DIR=/secure/storage/root-ca
   
   ./scripts/generate_root.sh
   ```

3. **Verify Generation**
   ```bash
   openssl x509 -in certs/root-ca.crt -text -noout
   openssl ec -in certs/root-ca.key -check
   ```

4. **Backup Root CA**
   - Create 3 encrypted copies
   - Store in separate physical locations
   - Use HSM or encrypted USB drives
   - Document locations in secure ledger

5. **Archive System**
   - Power down air-gapped system
   - Store securely
   - Document access procedures

### Root CA Key Storage

**Options** (in order of preference):

1. **Hardware Security Module (HSM)**
   - Recommended: Thales Luna HSM
   - FIPS 140-2 Level 3 certified
   - Multi-person authentication

2. **Encrypted Storage**
   - GPG-encrypted key files
   - Passphrase known only to key custodians
   - Stored offline in secure facility

3. **Key Splitting (Shamir's Secret Sharing)**
   - Split key into N shares
   - Require M of N shares to reconstitute
   - Example: 5 shares, require 3
   ```bash
   # Using ssss tool
   ssss-split -t 3 -n 5 < root-ca.key
   ```

### Root CA Usage

**Frequency**: Once per year (or less)

**Purpose**:
- Sign new intermediate CA certificates
- Revoke compromised intermediate CAs
- Update CRL

**Procedure**:
1. Retrieve root CA from secure storage
2. Boot air-gapped system
3. Perform signing operation
4. Generate new CRL if needed
5. Secure all outputs
6. Return root CA to storage
7. Log all operations in secure ledger

## Intermediate CA Operations

### Key Rotation

**Schedule**: Every 2 years

**Procedure**:

1. **Generate New Intermediate CA**
   ```bash
   # On air-gapped system with root CA
   gigvault ca create-intermediate \
     --root-cert /secure/root-ca.crt \
     --root-key /secure/root-ca.key \
     --cn "GigVault Intermediate CA 2025" \
     --validity 730  # 2 years
   ```

2. **Deploy New Intermediate CA**
   ```bash
   # Create Kubernetes secret
   kubectl create secret tls ca-cert-new \
     --cert=intermediate-ca-2025.crt \
     --key=intermediate-ca-2025.key \
     -n gigvault
   ```

3. **Overlap Period**
   - Run old and new CAs in parallel for 90 days
   - Issue new certificates from new CA
   - Old certificates remain valid until expiry

4. **Decommission Old CA**
   ```bash
   # After overlap period
   kubectl delete secret ca-cert-old -n gigvault
   kubectl rollout restart deployment/ca -n gigvault
   ```

### Emergency CA Replacement

**Scenario**: Intermediate CA key compromised

**Immediate Actions** (within 1 hour):

1. **Revoke Compromised CA**
   ```bash
   # Revoke intermediate CA at root
   gigvault ca revoke \
     --root-cert root-ca.crt \
     --root-key root-ca.key \
     --serial <intermediate-serial>
   ```

2. **Generate and Deploy New CA**
   ```bash
   # Generate new intermediate
   gigvault ca create-intermediate ...
   
   # Deploy immediately
   kubectl create secret tls ca-cert-emergency ...
   kubectl set env deployment/ca CA_SECRET=ca-cert-emergency -n gigvault
   ```

3. **Notify All Parties**
   - Internal teams
   - External certificate holders
   - Certificate transparency logs

4. **Revoke All Issued Certificates**
   ```bash
   # Mass revocation
   gigvault ca revoke-all-issued-by <compromised-ca-serial>
   ```

## Certificate Lifecycle Management

### Issuance

**Standard Process**:
1. CSR received → RA validates
2. RA approves → CA signs
3. Certificate issued → audit log
4. Notification sent to requester

**Auto-Renewal**:
- Enabled for ACME clients
- Renewal window: 30 days before expiry
- Automatic retry on failure

### Revocation

**Valid Reasons**:
- Key compromise
- CA compromise
- Affiliation changed
- Superseded
- Cessation of operation

**Procedure**:
```bash
# Via CLI
gigvault cert revoke <serial> --reason key-compromise

# Via API
curl -X POST https://ca.example.com/api/v1/certificates/<serial>/revoke \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"reason": "key-compromise"}'
```

**Propagation**:
- OCSP: Immediate (cache cleared)
- CRL: Next scheduled generation (hourly)
- Notifications: Sent within 5 minutes

### Monitoring

**Key Metrics**:

| Metric | Alert Threshold |
|--------|-----------------|
| Certificate issuance rate | > 1000/min (anomaly) |
| Failed issuance rate | > 5% |
| OCSP response time | > 100ms (p99) |
| Certificate expiry | < 7 days remaining |
| Revocation rate | > 10/min (anomaly) |

**Monitoring Tools**:
```bash
# Prometheus queries
rate(ca_certificates_issued_total[5m])
histogram_quantile(0.99, rate(ocsp_response_duration_seconds_bucket[5m]))
ca_certificates_expiring_soon{days="7"}
```

## Security Incidents

### Key Compromise Response

**Detection Indicators**:
- Unauthorized certificate issuance
- Unexpected key usage
- HSM tamper alerts
- Audit log anomalies

**Response Procedure**:

1. **Immediate** (< 1 hour):
   - Isolate compromised system
   - Revoke compromised key/certificate
   - Enable enhanced monitoring
   - Alert security team

2. **Short-term** (< 24 hours):
   - Generate new keys
   - Issue replacement certificates
   - Update CRLs and OCSP
   - Notify affected parties

3. **Long-term** (< 1 week):
   - Root cause analysis
   - Security audit
   - Update procedures
   - Report to management

### Audit Log Investigation

**Querying Audit Logs**:
```bash
# Find all actions by user
SELECT * FROM audit_events 
WHERE actor = 'alice@example.com' 
ORDER BY timestamp DESC;

# Find certificate issuances
SELECT * FROM audit_events 
WHERE action = 'certificate.issue' 
AND timestamp > NOW() - INTERVAL '24 hours';

# Verify log integrity
gigvault audit verify --from 2025-01-01 --to 2025-01-31
```

## Backup and Recovery

### Backup Schedule

| Component | Frequency | Retention |
|-----------|-----------|-----------|
| Database | Every 6 hours | 30 days |
| Configuration | Daily | 90 days |
| Audit logs | Real-time | 7 years |
| HSM backup | Weekly | Forever |

### Database Backup

```bash
# Backup PostgreSQL
kubectl exec -n gigvault postgresql-0 -- \
  pg_dump -U gigvault gigvault > backup-$(date +%Y%m%d).sql

# Encrypt backup
gpg --encrypt --recipient ops@example.com backup-*.sql

# Upload to S3
aws s3 cp backup-*.sql.gpg s3://gigvault-backups/
```

### Disaster Recovery

**RTO**: 15 minutes
**RPO**: 5 minutes

**Recovery Procedure**:

1. **Restore Database**
   ```bash
   # From latest backup
   kubectl exec -i postgresql-0 -n gigvault -- \
     psql -U gigvault gigvault < backup-latest.sql
   ```

2. **Restore Secrets**
   ```bash
   # From Vault/Secrets Manager
   kubectl create secret tls ca-cert \
     --cert=<restored-cert> \
     --key=<restored-key>
   ```

3. **Redeploy Services**
   ```bash
   helm upgrade --install ca charts/ca -n gigvault
   ```

4. **Verify Recovery**
   ```bash
   make smoke-test
   ```

## Access Control

### Role-Based Access

| Role | Permissions |
|------|-------------|
| **Root Admin** | All operations, including root CA access |
| **CA Operator** | Issue/revoke certificates, manage policies |
| **RA Approver** | Approve/reject CSRs |
| **Auditor** | Read-only access to logs |
| **Developer** | Issue dev certificates only |

### MFA Requirements

- Root CA operations: Hardware token + passphrase
- Production CA: TOTP or hardware token
- RA approvals: TOTP
- Audit log access: TOTP
- Admin UI: TOTP or WebAuthn

### Access Audit

```bash
# Review access logs
SELECT actor, action, resource, timestamp 
FROM audit_events 
WHERE action LIKE 'auth.%' 
ORDER BY timestamp DESC 
LIMIT 100;

# Failed auth attempts
SELECT actor, COUNT(*) 
FROM audit_events 
WHERE action = 'auth.failed' 
AND timestamp > NOW() - INTERVAL '24 hours' 
GROUP BY actor 
HAVING COUNT(*) > 5;
```

## Compliance

### Audit Requirements

- **SOC 2**: Comprehensive logging, access controls
- **PCI DSS**: Key protection, change management
- **ISO 27001**: Information security management
- **WebTrust**: CA/Browser Forum baseline requirements

### Reporting

**Monthly Report Includes**:
- Certificates issued/revoked
- Incident summary
- System uptime
- Security patches applied
- Access log review

**Annual Report Includes**:
- Full security audit
- Key rotation summary
- DR test results
- Compliance certification status

## Contact Information

### Emergency Contacts

- **Security Incidents**: security@gigvault.io (24/7)
- **Operations**: ops@gigvault.io
- **On-Call Engineer**: +1-555-PKI-ONCALL

### Escalation Path

1. L1: Operations team
2. L2: Security team
3. L3: CTO / CISO
4. Emergency: CEO

---

**Last Updated**: 2025-01-01
**Next Review**: 2025-07-01
**Owner**: Security Operations Team

