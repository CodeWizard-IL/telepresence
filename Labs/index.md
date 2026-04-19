# Telepresence Labs :satellite:

A comprehensive collection of hands-on labs for learning and mastering **Telepresence** — the tool that bridges your local development environment with a remote Kubernetes cluster.

---

## Overview

These labs provide a progressive learning path from basic cluster connectivity to advanced debugging techniques. Each lab builds upon the previous one, taking you from Telepresence fundamentals to production-grade workflows.

---

## :rocket: Lab Series

| Lab | Title | Description |
|-----|-------|-------------|
| 001 | [The Cluster Bridge](./001-cluster-bridge/) | Connectivity & DNS — access cluster services from your local machine |
| 002 | [The Inner Dev Loop](./002-inner-dev-loop/) | Global Intercept — route all service traffic to your local code |
| 003 | [The Ghost Developer](./003-ghost-developer/) | Personal Intercepts — develop without affecting teammates |
| 004 | [The Context Clone](./004-context-clone/) | Env Vars & Secrets — run locally with remote configuration |
| 005 | [The Deep Dive](./005-deep-dive/) | Volume Mounts & Debugging — mount remote files and live debug |

---

## :wrench: Prerequisites

Before starting these labs, ensure you have:

- A Kubernetes cluster (k3s, EKS, GKE, Minikube, etc.)
- `kubectl` configured to access the cluster
- Telepresence CLI installed
- Python 3.8+ (for sample applications)
- A text editor or IDE (VS Code recommended)

---

## :bulb: How to Use These Labs

1. **Start with Lab 001**: Establish connectivity to your cluster
2. **Follow the sequence**: Each lab builds on the previous one
3. **Run the commands**: Every lab includes copy-paste ready commands
4. **Experiment**: Modify the examples to match your own services
