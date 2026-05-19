# platform-lab

Production-style GitOps Kubernetes lab built for experimenting with platform engineering, deployment automation, ingress, TLS, observability, and internal developer platform patterns.

## Goals

This repository is used to experiment with and document:

- Kubernetes
- GitOps workflows
- Argo CD
- ingress-nginx
- cert-manager
- TLS automation
- Production-style infrastructure patterns
- Internal platform engineering concepts
- Deployment automation
- Cloud-native application delivery

---

## Current Stack

### Infrastructure

- Vultr Kubernetes Engine (VKE)
- Kubernetes v1.35
- ingress-nginx
- cert-manager
- Let’s Encrypt
- Argo CD

### GitOps

- Argo CD
- Kustomize
- GitHub

---

## Repository Structure

```text
.
├── docs/
├── manifests/
│   └── staging/
├── README.md
└── .gitignore
```

---

## Cluster Environments

| Environment | Purpose |
|---|---|
| staging | Main experimentation and platform learning cluster |

---

## Core Components

| Component | Namespace |
|---|---|
| ingress-nginx | ingress |
| cert-manager | cert-manager |
| Argo CD | argocd |

---

## Features Implemented

- Kubernetes cluster provisioning on Vultr
- ingress-nginx installation
- Argo CD installation
- HTTPS/TLS with cert-manager
- Automated Let’s Encrypt certificates
- GitOps-ready repository structure
- Production-style ingress architecture

---

## GitOps Philosophy

The goal of this repository is to move toward a fully declarative GitOps workflow where:

```text
GitHub → Argo CD → Kubernetes Cluster
```

becomes the single source of truth.

---

## Future Plans

- Argo CD self-management
- Application-of-apps pattern
- Monitoring stack
- Logging stack
- External DNS
- Sealed Secrets / External Secrets
- CI/CD pipelines
- Multi-environment overlays
- Internal developer platform patterns

---

## Documentation

Detailed setup and operational documentation lives under:

```text
docs/
```

Including:

- Cluster bootstrap
- Argo CD setup
- Ingress and TLS
- GitOps workflows
- Troubleshooting

---

## License

MIT
