# Platform Charts

This repository contains a collection of "Wrapper" Helm charts used to bootstrap a standardized environment on Kubernetes or OpenShift for our POC and Demo projects.

The goal of this repository is to provide a consistent "Landing Zone" for applications, ensuring they have immediate access to various application dealing with cross cutting concerns such as security, config / secret management, database services, etc. regardless of the underlying Kubernetes cluster type.

## Platform Architecture

The following diagram shows how the components interact to create a secure, automated environment. Even though this is a Demo/POC environment, it utilizes mTLS, Certificate Rotation, Secret Orchestration, and various other concepts to mimic a production-grade architecture.

<!-- Add Diagram Here -->


## Components Catalog
To ensure a successful deployment, we follow the recommended installation sequence documented within each chart's readme.

S.No | Component	| Description	| Status
| --- | --- | --- | --- |
1 | [Cert-Manager](./cert-manager/) | Automates TLS certificate issuance and renewal.	| ✅ Ready
2 | [HashiCorp Vault](./hashicorp-vault/)	| Centralized secret management and dynamic credential generation / rotation. | ✅ Ready
3 | [External Secrets](./external-secrets/)	| Syncs secrets from Vault into native Kubernetes Secrets. | ✅ Ready
4 | [PostgreSQL](./postgres/) | Secure, TLS-enabled database with Vault credentail creation / rotation. | ✅ Ready
5 | [Stakater Reloader](./skater-reloader/)	| Triggers automatic app restarts when Secrets/ConfigMaps change so that the new config / secret values are loaded into the app. | ✅ Ready
6 | [Headlamp](./headlamp/)	| Modern Kubernetes UI (Dashboard) required only for standard Kubernetes clusters. For OpenShift the native UI is used.	| ✅ Ready
7 | [Kafka](./kafka/)	| Distributed event streaming platform for high-performance data pipelines.	| ❌ To Do

## Getting Started

Prepare your Cluster: Ensure you have a running Kubernetes or OpenShift cluster. Refer to the [infra-setup](https://github.com/Microservices-Demo-Projects/infra-setup) repository for setup instructions (Local OpenShift - CRC / Kubernetes - Kind; Cloud - EKS, etc.).

Install Tools: Ensure you have `helm`, `kubectl`, and `oc` (if using OpenShift) installed.

Deploy Components: Navigate to the individual directories above and follow the `README.md` instructions for each component in the suggested order.

> [!WARNING]
> This repository is intended for Demo and POC purposes. While it follows several production ready security best practices, always review configurations before using in a production / critical environment.