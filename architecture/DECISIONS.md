# Technical Decision Log

Record of significant technical decisions made during GigVault development.

## Format

Each decision follows this structure:
- **ID**: Unique identifier
- **Date**: When the decision was made
- **Status**: Proposed, Accepted, Deprecated, Superseded
- **Context**: What situation led to this decision
- **Decision**: What we decided to do
- **Consequences**: What are the trade-offs

---

## ADR-001: Use ECDSA Instead of RSA

**Date**: 2024-12-01
**Status**: Accepted

**Context**:
We needed to choose a default key algorithm for the PKI system. The main candidates were RSA and ECDSA.

**Decision**:
Use ECDSA (P-256 and P-384 curves) as the default key algorithm for all certificates, with RSA support as optional for legacy compatibility.

**Rationale**:
- **Performance**: ECDSA is significantly faster for signing operations
- **Key Size**: 256-bit ECDSA ≈ 3072-bit RSA security, but much smaller
- **Modern**: Industry trend is moving toward ECDSA
- **TLS 1.3**: Better support for ECDSA
- **HSM**: Modern HSMs have excellent ECDSA support

**Consequences**:
- **Positive**: Faster operations, smaller certificates, lower bandwidth
- **Negative**: Some very old clients don't support ECDSA
- **Mitigation**: Maintain RSA support for legacy systems

---

## ADR-002: Microservices Architecture

**Date**: 2024-12-01
**Status**: Accepted

**Context**:
We needed to decide between a monolithic application or microservices for the PKI platform.

**Decision**:
Implement as microservices with each major function (CA, RA, OCSP, etc.) as a separate service.

**Rationale**:
- **Scalability**: Scale individual services independently (OCSP needs more instances than CA)
- **Security**: Isolation between components reduces blast radius
- **Development**: Teams can work independently on different services
- **Resilience**: Failure of one service doesn't bring down the entire system
- **Technology**: Can use different storage/tech for different services if needed

**Consequences**:
- **Positive**: Better scalability, security, and team autonomy
- **Negative**: More operational complexity, inter-service communication overhead
- **Mitigation**: Use Kubernetes for orchestration, implement service mesh for secure communication

---

## ADR-003: PostgreSQL as Primary Database

**Date**: 2024-12-05
**Status**: Accepted

**Context**:
Needed to choose a database for storing certificates, CSRs, and metadata.

**Decision**:
Use PostgreSQL for all relational data storage needs.

**Rationale**:
- **ACID**: Strong consistency guarantees for critical PKI data
- **JSON Support**: Can store flexible metadata
- **Performance**: Excellent for read-heavy workloads (certificate lookups)
- **Replication**: Built-in streaming replication for HA
- **Maturity**: Battle-tested, well-understood
- **Open Source**: No vendor lock-in

**Alternatives Considered**:
- **MySQL**: Less robust JSON support
- **MongoDB**: Too flexible, ACID not guaranteed
- **SQLite**: Not suitable for distributed systems

**Consequences**:
- **Positive**: Reliable, performant, good tooling
- **Negative**: Requires ops expertise for tuning and HA
- **Mitigation**: Use managed PostgreSQL in production (RDS, CloudSQL, etc.)

---

## ADR-004: Go as Primary Language

**Date**: 2024-12-01
**Status**: Accepted

**Context**:
Needed to choose a programming language for all backend services.

**Decision**:
Use Go 1.23 for all microservices.

**Rationale**:
- **Performance**: Compiled, fast execution
- **Concurrency**: Excellent goroutine support for handling many requests
- **Cryptography**: Strong crypto library support
- **Deployment**: Single binary, easy to containerize
- **Team Expertise**: Team has Go experience
- **Ecosystem**: Great tooling, libraries, and community

**Consequences**:
- **Positive**: Fast, reliable, easy to deploy
- **Negative**: More verbose than Python/Ruby, generics still maturing
- **Trade-off**: Accepted for the performance and reliability benefits

---

## ADR-005: Kubernetes for Orchestration

**Date**: 2024-12-10
**Status**: Accepted

**Context**:
Needed to choose a container orchestration platform.

**Decision**:
Use Kubernetes for all deployments, with kind for local development.

**Rationale**:
- **Industry Standard**: Widely adopted, lots of tooling
- **Portability**: Run anywhere (cloud, on-prem)
- **Features**: Auto-scaling, self-healing, service discovery
- **Operator Pattern**: Can build custom operators for CA lifecycle
- **Community**: Large ecosystem and community support

**Alternatives Considered**:
- **Docker Swarm**: Simpler but less feature-rich
- **Nomad**: Good but smaller ecosystem
- **ECS/Fargate**: AWS-specific, vendor lock-in

**Consequences**:
- **Positive**: Feature-rich, portable, well-supported
- **Negative**: Steep learning curve, operational complexity
- **Mitigation**: Use managed Kubernetes (EKS, GKE, AKS) in production

---

## ADR-006: Offline Root CA

**Date**: 2024-12-01
**Status**: Accepted

**Context**:
Needed to decide whether the root CA should be online or offline.

**Decision**:
Root CA is **always offline**, stored on air-gapped systems. Only intermediate CAs are online.

**Rationale**:
- **Security**: Root CA compromise is catastrophic
- **Best Practice**: Industry standard (Let's Encrypt, DigiCert, etc. all do this)
- **Compliance**: Required for WebTrust and other certifications
- **Risk Reduction**: Minimize exposure of root key

**Consequences**:
- **Positive**: Maximum security for root CA
- **Negative**: Operational overhead for root CA operations (signing intermediates)
- **Trade-off**: Security benefits far outweigh operational complexity

---

## ADR-007: ACME Protocol for Automated Enrollment

**Date**: 2024-12-15
**Status**: Accepted

**Context**:
Needed to choose protocol(s) for automated certificate enrollment.

**Decision**:
Implement ACME (Automatic Certificate Management Environment) as the primary automated enrollment protocol, with EST and SCEP for legacy support.

**Rationale**:
- **Standard**: RFC 8555, widely adopted
- **Automation**: Designed for automation, no human intervention
- **Compatibility**: Let's Encrypt clients (certbot, acme.sh) work out-of-box
- **Validation**: Multiple challenge types (HTTP-01, DNS-01, TLS-ALPN-01)
- **Industry**: Becoming de facto standard

**Consequences**:
- **Positive**: Easy integration, automation-first
- **Negative**: More complex than simple API
- **Mitigation**: Provide CLI and SDK for common use cases

---

## ADR-008: Helm for Kubernetes Deployments

**Date**: 2024-12-20
**Status**: Accepted

**Context**:
Needed a way to package and deploy Kubernetes applications.

**Decision**:
Use Helm charts for all service deployments.

**Rationale**:
- **Templating**: Parameterize manifests for different environments
- **Versioning**: Track deployment versions
- **Rollback**: Easy rollback to previous versions
- **Community**: Large chart repository, well-understood

**Alternatives Considered**:
- **Kustomize**: Good but less flexible templating
- **Raw YAML**: No templating or versioning
- **Operators**: Overkill for most services

**Consequences**:
- **Positive**: Flexible, versioned deployments
- **Negative**: Another tool to learn
- **Trade-off**: Benefits outweigh learning curve

---

## ADR-009: Structured Logging with Zap

**Date**: 2024-12-05
**Status**: Accepted

**Context**:
Needed a logging strategy for all services.

**Decision**:
Use structured JSON logging with Uber's Zap library.

**Rationale**:
- **Performance**: Zap is extremely fast
- **Structured**: JSON output easy to parse and query
- **Levels**: Debug, Info, Warn, Error support
- **Context**: Can attach structured context to logs
- **Ecosystem**: Integrates with Loki, ELK, etc.

**Consequences**:
- **Positive**: Fast, queryable, structured logs
- **Negative**: JSON less readable for humans
- **Mitigation**: Use console encoder for local development

---

## ADR-010: Monorepo vs Multi-Repo

**Date**: 2024-12-01
**Status**: Multi-Repo (Accepted)

**Context**:
Needed to decide repository structure for 15+ services.

**Decision**:
Use multiple repositories (one per service) under the `github.com/gigvault/` organization.

**Rationale**:
- **Independence**: Services can evolve independently
- **Access Control**: Granular permissions per repo
- **CI/CD**: Simpler per-service pipelines
- **Clarity**: Clear ownership and boundaries

**Alternatives Considered**:
- **Monorepo**: Easier to refactor across services, but CI/CD more complex

**Consequences**:
- **Positive**: Clear boundaries, independent releases
- **Negative**: Harder to make cross-service changes
- **Mitigation**: Use shared library (`gigvault/shared`) for common code

---

## ADR-011: No External Secrets in Code

**Date**: 2024-12-01
**Status**: Accepted

**Context**:
How to handle secrets (DB passwords, API keys, CA keys).

**Decision**:
- **Development**: Kubernetes secrets with placeholder values
- **Production**: HashiCorp Vault or cloud secret managers
- **Never**: Commit secrets to Git

**Rationale**:
- **Security**: Prevent secret leakage
- **Rotation**: Vault enables automated rotation
- **Audit**: Track secret access
- **Compliance**: Required for SOC 2, ISO 27001

**Consequences**:
- **Positive**: Secure secret management
- **Negative**: Additional infrastructure (Vault)
- **Mitigation**: Use managed solutions (AWS Secrets Manager, GCP Secret Manager)

---

## ADR-012: Immutable Audit Logs

**Date**: 2024-12-15
**Status**: Accepted

**Context**:
Audit logs must be tamper-proof for compliance.

**Decision**:
Audit logs are cryptographically signed and stored in append-only fashion. Each log entry includes a signature chain from the previous entry.

**Rationale**:
- **Compliance**: Required for WebTrust, SOC 2
- **Tamper Detection**: Any modification is detectable
- **Non-Repudiation**: Signed logs provide proof of actions

**Consequences**:
- **Positive**: Tamper-proof logs, compliance-ready
- **Negative**: Cannot delete logs (must retain for compliance periods)
- **Mitigation**: Automated archival to cold storage after retention period

---

**Document Owner**: Architecture Team
**Last Updated**: 2025-01-01
**Next Review**: Quarterly

