# GigVault Documentation

Complete documentation for the GigVault PKI ecosystem.

## Table of Contents

### Architecture
- [System Architecture](architecture/ARCHITECTURE.md) - High-level system design
- [Service Topology](architecture/TOPOLOGY.md) - Service relationships and communication
- [Security Model](architecture/SECURITY.md) - Security architecture and threat model
- [Decision Log](architecture/DECISIONS.md) - Technical decisions and rationale

### Guides
- [Local Bootstrap](guides/BOOTSTRAP_LOCAL.md) - Set up local development environment
- [Production Deployment](guides/DEPLOYMENT.md) - Deploy to production
- [Operations Manual](guides/OPSEC.md) - Day-to-day operations and procedures
- [Disaster Recovery](guides/DISASTER_RECOVERY.md) - Recovery procedures

### API Documentation
- [CA API](api/CA_API.md) - Certificate Authority REST API
- [RA API](api/RA_API.md) - Registration Authority REST API
- [CLI Reference](api/CLI.md) - Command-line interface

## Quick Start

```bash
# Clone repositories
git clone https://github.com/gigvault/infra.git

# Bootstrap local environment
cd infra
make kind-up
make build-all
make deploy-local
make smoke-test
```

## License

Copyright © 2025 GigVault

