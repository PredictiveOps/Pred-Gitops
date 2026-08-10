# Predictive Maintenance GitOps Repository

This repository contains the Kubernetes deployment state for the Predictive Maintenance system. It is the GitOps source of truth watched by Argo CD in the live Kubernetes cluster.

The application source code lives in the developer repository: [PredictiveOps/Pred](https://github.com/PredictiveOps/Pred). This repository focuses on the DevOps side of the project: Kubernetes manifests, service configuration, ingress routing, observability, persistent infrastructure, and Argo CD synchronization.

## DevOps Workflow

The project follows a GitOps workflow where application code and cluster state are separated:

1. Developers work on services in [`PredictiveOps/Pred`](https://github.com/PredictiveOps/Pred).
2. Service images are built and pushed to Google Artifact Registry.
3. Kubernetes manifests in this repository are updated with the required image tags and configuration changes.
4. Argo CD watches this repository and continuously compares it with the live cluster state.
5. When a change is merged into this GitOps repository, Argo CD automatically syncs it into the `pred` namespace.
6. Argo CD self-healing keeps the cluster aligned with the desired state declared in Git.

The Argo CD application is defined in [`argocd/application.yaml`](argocd/application.yaml). It points to this repository, recursively applies manifests from the [`apps`](apps) directory, and enables automated sync with pruning and self-healing.

```yaml
source:
  repoURL: https://github.com/PredictiveOps/Pred-Gitops
  targetRevision: HEAD
  path: apps
  directory:
    recurse: true
destination:
  namespace: pred
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

## Kubernetes Cluster Design

The system is deployed as a set of containerized services running inside Kubernetes. The cluster contains application services, messaging infrastructure, storage, authentication, API gateway routing, ingress, TLS, and monitoring.

![Kubernetes cluster design](images/Kube%20cluster%20design.png)

## Deployed Architecture

The Kubernetes manifests define the following major parts of the platform:

| Area | Kubernetes Resources | Purpose |
| --- | --- | --- |
| Frontend | `web-frontend` Deployment and Service | Hosts the user-facing web application. |
| API Gateway | `kong` Deployment and Service | Routes API traffic, applies JWT validation, CORS, rate limiting, and request-size limits. |
| Authentication | `keycloak` Deployment and Service | Provides identity and access management for the platform. |
| Ingestion | `ingestion-service` Deployment and Service | Receives device data through MQTT and publishes events into Kafka. |
| Event Processing | `event-processing-service` Deployment and Service | Consumes device events and processes them for downstream services. |
| ML Service | `ml-service` Deployment and Service | Handles predictive maintenance model workloads and consumes ML feature events. |
| Notifications | `notifications-service` Deployment and Service | Produces notifications from processed events and prediction outcomes. |
| Messaging | `kafka` Deployment and Service | Event streaming backbone for communication between services. |
| MQTT Broker | `mosquitto` Deployment, Services, and ConfigMap | Handles device telemetry and registration topics. |
| Cache | `redis` Deployment and Service | Shared caching layer used by backend services. |
| Database | `postgres` Deployment, Service, and PVC | Persistent relational storage for application services and Keycloak. |
| Observability | `prometheus` Deployment, Service, and ConfigMap | Collects metrics from the cluster and application workloads. |
| Edge Routing | GKE Ingress and ManagedCertificate | Exposes the platform through HTTPS routes. |
| TLS Issuer | cert-manager `ClusterIssuer` | Provides certificate automation support. |

## Request Routing

External traffic is managed through a GKE ingress resource declared in [`apps/ingress/ingress.yaml`](apps/ingress/ingress.yaml). The ingress uses a static global IP, a managed certificate, and HTTPS-only routing to expose the web application, API gateway, and authentication service.

The routing layer separates user-facing traffic, API traffic, and authentication traffic:

| Route Purpose | Routed Service | Role |
| --- | --- | --- |
| Main web application | `web-frontend` | Serves the frontend experience. |
| API requests | `kong` | Routes backend API calls through the API gateway. |
| Authentication paths | `keycloak` | Handles login, realm, admin, and identity-provider resources. |

Kong is configured declaratively through the `kong-config` ConfigMap in [`apps/shared/configmap.yaml`](apps/shared/configmap.yaml). It routes protected API paths to backend services and uses JWT validation against Keycloak.

## Event-Driven Data Flow

The backend is organized around event-driven communication:

1. Devices publish telemetry and registration messages to Mosquitto over MQTT.
2. The ingestion service consumes MQTT messages, validates them, and publishes normalized events to Kafka.
3. Event processing services consume Kafka topics and persist processed data to PostgreSQL.
4. The ML service consumes feature events and stores prediction-related results.
5. The notifications service consumes notification events and integrates with the authentication context from Keycloak.
6. The web frontend and API gateway expose the processed data and platform capabilities to users.

This structure keeps the services independently deployable while allowing asynchronous communication between device ingestion, processing, prediction, and notification workflows.

## Repository Structure

```text
.
+-- apps/
|   +-- ai-ml/                         # ML service deployment and service
|   +-- api-gateway/                   # Kong API gateway and JWKS sync sidecar
|   +-- event-processing-service/      # Event processing workload
|   +-- ingestion-service/             # MQTT-to-Kafka ingestion workload
|   +-- ingress/                       # GKE ingress and managed certificate
|   +-- kafka/                         # Kafka deployment and service
|   +-- keycloak/                      # Identity provider deployment and services
|   +-- mosquitto/                     # MQTT broker configuration and services
|   +-- notifications-service/         # Notification workload
|   +-- observability/
|   |   +-- prometheus/                # Prometheus monitoring resources
|   +-- postgres/                      # PostgreSQL deployment, service, and PVC
|   +-- redis/                         # Redis deployment and service
|   +-- shared/                        # Shared ConfigMaps and Secrets
|   +-- web-frontend/                  # Frontend deployment and service
+-- argocd/
|   +-- application.yaml               # Argo CD application definition
+-- cert-manager/
|   +-- clusterissuer.yaml             # Cluster-wide certificate issuer
+-- images/
    +-- Kube cluster design.png        # Kubernetes cluster architecture diagram
```

## GitOps Highlights

- **Declarative infrastructure:** The desired Kubernetes state is versioned as YAML manifests.
- **Automated delivery:** Argo CD syncs committed changes from this repository into the live cluster.
- **Self-healing cluster state:** Manual drift in the cluster is corrected by Argo CD.
- **Pruning enabled:** Removed manifests are also removed from the cluster during sync.
- **Separated concerns:** Application code is maintained in the developer repository, while deployment state is maintained here.
- **Production-style routing:** GKE Ingress, managed certificates, and Kong API Gateway expose services through HTTPS domains.
- **Service-oriented architecture:** Frontend, authentication, gateway, ingestion, event processing, ML, notifications, messaging, cache, database, and observability are deployed as separate Kubernetes workloads.

## Technology Stack Represented Here

- Kubernetes
- Argo CD
- Google Kubernetes Engine ingress
- Google Artifact Registry container images
- Kong Gateway
- Keycloak
- Kafka
- Eclipse Mosquitto
- PostgreSQL
- Redis
- Prometheus
- cert-manager
