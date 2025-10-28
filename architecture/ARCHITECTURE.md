# GigVault System Architecture

## Overview

GigVault is a modern, cloud-native Public Key Infrastructure (PKI) platform built on microservices architecture, designed for scalability, security, and operational excellence.

## Core Principles

1. **Separation of Concerns**: Each service has a single, well-defined responsibility
2. **Zero Trust**: All inter-service communication uses mTLS
3. **Immutability**: Audit logs and certificates are immutable
4. **Scalability**: Horizontal scaling for all online services
5. **Security by Default**: ECDSA P-256/P-384, secure defaults, least privilege

## System Components

### Core PKI Services

#### Root CA (`rootca`)
- **Purpose**: Offline root certificate authority
- **Key Algorithm**: ECDSA P-384
- **Operations**: Air-gapped, manual operations only
- **Lifecycle**: 20-year validity, rarely used

#### Intermediate CA (`ca`)
- **Purpose**: Online certificate issuance
- **Key Algorithm**: ECDSA P-256/P-384
- **Operations**: Automated signing, renewal, revocation
- **Database**: PostgreSQL (certificates, metadata)
- **API**: REST + gRPC

#### Registration Authority (`ra`)
- **Purpose**: Enrollment validation and approval
- **Workflow**: CSR submission → manual/auto approval → forward to CA
- **Database**: PostgreSQL (enrollments)
- **Integration**: Webhooks for approvals

### Key Management

#### Key Manager (`keymgr`)
- **Purpose**: Centralized key lifecycle management
- **Features**: 
  - Key generation (ECDSA, RSA)
  - HSM integration (PKCS#11)
  - Key rotation
  - Secure storage
- **Storage**: Encrypted secrets in Kubernetes, HSM for production

### Enrollment Protocols

#### Enrollment Service (`enroll`)
- **Protocols**:
  - **ACME**: Let's Encrypt-compatible
  - **EST**: RFC 7030
  - **SCEP**: Legacy systems
- **Features**: Automated certificate enrollment
- **Use Cases**: Web servers, IoT devices, mobile clients

### Revocation Services

#### OCSP Responder (`ocsp`)
- **Purpose**: Real-time certificate status checks
- **Protocol**: RFC 6960 (OCSP)
- **Performance**: In-memory cache, Redis backing
- **SLA**: < 100ms response time

#### CRL Generator (`crl`)
- **Purpose**: Certificate Revocation List generation
- **Schedule**: Hourly generation, daily full CRL
- **Distribution**: HTTP, LDAP endpoints
- **Format**: X.509 CRL v2

### Policy & Security

#### Policy Engine (`policy`)
- **Purpose**: Certificate issuance policies
- **Features**:
  - Subject validation
  - Key usage restrictions
  - Validity period limits
  - Domain whitelisting/blacklisting
- **Configuration**: YAML-based policies

#### Auth Service (`auth`)
- **Purpose**: Authentication and authorization
- **Protocols**: OIDC, OAuth 2.0, JWT
- **Features**:
  - User management
  - Role-based access control (RBAC)
  - API key management
- **Integration**: External IdPs (Okta, Auth0, Keycloak)

#### Audit Service (`audit`)
- **Purpose**: Immutable audit logging
- **Features**:
  - Cryptographically signed logs
  - Tamper detection
  - Compliance reporting
- **Storage**: PostgreSQL + S3 archive
- **Retention**: 7 years (configurable)

#### Notification Service (`notify`)
- **Purpose**: Certificate lifecycle notifications
- **Channels**: Email, Slack, webhooks, PagerDuty
- **Events**: 
  - Certificate expiry warnings (90d, 30d, 7d)
  - Revocation alerts
  - Failed enrollment attempts

### Developer Tools

#### CLI (`cli`)
- **Purpose**: Operator command-line interface
- **Commands**:
  ```bash
  gigvault cert create <cn>
  gigvault csr submit <file>
  gigvault key generate <name>
  gigvault cert revoke <serial>
  ```

#### Admin UI (`admin-ui`)
- **Technology**: React + Next.js
- **Features**:
  - Certificate dashboard
  - CSR approval workflow
  - Audit log viewer
  - User management

### Infrastructure

#### Kubernetes Operator (`operator`)
- **Purpose**: Kubernetes-native CA management
- **CRDs**:
  - `CertificateAuthority`
  - `Certificate`
  - `Policy`
- **Features**: Auto-renewal, secret management

## Data Flow

### Certificate Issuance Flow

```
Client → enroll (ACME/EST/SCEP)
         ↓
       ra (validation)
         ↓ (if approved)
       ca (signing)
         ↓
    Certificate → Client
         ↓
    audit (logging)
         ↓
    notify (confirmation)
```

### Certificate Revocation Flow

```
Operator → ca /certificates/{serial}/revoke
            ↓
          audit (log)
            ↓
          ocsp (update cache)
            ↓
          crl (schedule regeneration)
            ↓
          notify (alert)
```

## Security Architecture

### Trust Model

```
Root CA (offline, air-gapped)
  ↓ signs
Intermediate CA (online, HSM-backed)
  ↓ signs
End-Entity Certificates
```

### mTLS Between Services

All inter-service communication uses mutual TLS:
- Each service has its own certificate
- Issued by the internal intermediate CA
- Auto-rotated every 90 days

### Secret Management

- **Development**: Kubernetes secrets
- **Production**: HashiCorp Vault / AWS Secrets Manager
- **HSM**: PKCS#11 for CA private keys

## Deployment Topology

### Development (kind)

```
kind cluster
  └─ gigvault namespace
      ├─ postgresql (single instance)
      ├─ ca (1 replica)
      ├─ ra (1 replica)
      ├─ keymgr (1 replica)
      └─ ... (other services)
```

### Production (Kubernetes)

```
Cluster 1 (Primary)
  ├─ PostgreSQL (HA, 3 replicas)
  ├─ Redis (HA, Sentinel)
  ├─ ca (3 replicas, load balanced)
  ├─ ra (2 replicas)
  ├─ ocsp (5 replicas, auto-scale)
  └─ ...

Cluster 2 (DR)
  └─ Standby replicas
```

## Observability

### Metrics
- **Tool**: Prometheus + Grafana
- **Metrics**:
  - Certificate issuance rate
  - OCSP response latency
  - Error rates
  - Queue depths

### Logging
- **Tool**: Loki / ELK
- **Structured**: JSON logs with correlation IDs
- **Retention**: 30 days hot, 1 year warm

### Tracing
- **Tool**: Jaeger / Tempo
- **Propagation**: W3C Trace Context

## Scalability

| Service | Scaling Strategy | Target Capacity |
|---------|------------------|-----------------|
| CA      | Horizontal       | 1000 certs/sec  |
| RA      | Horizontal       | 100 CSRs/sec    |
| OCSP    | Horizontal       | 10k req/sec     |
| CRL     | Vertical         | 1M certs/CRL    |
| Enroll  | Horizontal       | 500 req/sec     |

## High Availability

- **RTO**: Recovery Time Objective = 15 minutes
- **RPO**: Recovery Point Objective = 5 minutes
- **Availability Target**: 99.9% (three nines)

### Failure Scenarios

1. **Single Service Failure**: Auto-restart, health checks
2. **Database Failure**: Automatic failover to replica
3. **Cluster Failure**: Failover to DR cluster
4. **Root CA Compromise**: See OPSEC.md for recovery procedures

## Technology Stack

| Component | Technology |
|-----------|------------|
| Language  | Go 1.23    |
| Database  | PostgreSQL 16 |
| Cache     | Redis 7    |
| Container | Docker     |
| Orchestration | Kubernetes 1.29 |
| Service Mesh | Istio (optional) |
| Secrets   | HashiCorp Vault |
| HSM       | Thales Luna / AWS CloudHSM |

## Next Steps

- [Local Bootstrap Guide](../guides/BOOTSTRAP_LOCAL.md)
- [Production Deployment](../guides/DEPLOYMENT.md)
- [Operations Manual](../guides/OPSEC.md)

