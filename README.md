# PromptForge

PromptForge is an AI-powered application builder that turns natural-language prompts into working web applications. It uses a distributed Spring Boot architecture and isolated Kubernetes preview environments to generate, store, and run projects securely.

> Prompt, generate, preview, refine, and publish—all from one workspace.

## Features

- AI-assisted React application generation
- Live project previews using isolated Kubernetes runner pods
- Project and source-file management
- JWT-based authentication and authorization
- Real-time asynchronous processing with Apache Kafka
- S3-compatible project storage using MinIO
- Dynamic preview routing through Redis
- Automatic cleanup of expired preview environments
- Containerized microservices deployed to Google Kubernetes Engine
- Automated builds and deployments with GitHub Actions

## Architecture

```text
                         ┌──────────────────────┐
                         │   PromptForge UI     │
                         └──────────┬───────────┘
                                    │
                         ┌──────────▼───────────┐
                         │     API Gateway      │
                         └──────────┬───────────┘
                                    │
             ┌──────────────────────┼──────────────────────┐
             │                      │                      │
   ┌─────────▼─────────┐  ┌────────▼─────────┐  ┌────────▼──────────┐
   │ Account Service   │  │ Workspace Service│  │ Intelligence      │
   │ Auth and Accounts │  │ Projects/Previews│  │ Service           │
   └───────────────────┘  └────────┬─────────┘  └───────────────────┘
                                    │
                  ┌─────────────────┼─────────────────┐
                  │                 │                 │
          ┌───────▼───────┐ ┌──────▼──────┐ ┌────────▼────────┐
          │ PostgreSQL    │ │ MinIO       │ │ Kafka / Redis   │
          │               │ │ File Store  │ │ Events/Routing  │
          └───────────────┘ └─────────────┘ └─────────────────┘
                                    │
                         ┌──────────▼───────────┐
                         │ Kubernetes Preview  │
                         │ Runner Pool          │
                         └──────────┬───────────┘
                                    │
                         ┌──────────▼───────────┐
                         │ Wildcard Proxy      │
                         │ *.previews.domain   │
                         └──────────────────────┘
```

## How previews work

1. A user requests a preview for a project.
2. The workspace service claims an idle Kubernetes runner.
3. Project files are synchronized from MinIO.
4. Project dependencies are installed.
5. Vite starts inside the isolated runner container.
6. The service waits until the preview is healthy.
7. Redis maps the project hostname to the runner pod.
8. The wildcard proxy forwards HTTP and WebSocket traffic.
9. Expired preview pods and routes are removed automatically.

The runner pool maintains two idle pods so new previews can start without waiting for Kubernetes to provision an entirely new environment.

## Technology stack

| Area | Technology |
|---|---|
| Backend | Java 21, Spring Boot, Spring Cloud |
| Frontend projects | React, TypeScript, Vite |
| API routing | Spring Cloud Gateway |
| Service communication | REST, OpenFeign, Kafka |
| Authentication | Spring Security, JWT |
| Database | PostgreSQL, pgvector |
| Object storage | MinIO |
| Cache and preview routing | Redis |
| Service discovery | Netflix Eureka |
| Containers | Docker, Jib |
| Orchestration | Kubernetes, Google Kubernetes Engine |
| Reverse proxy | Node.js, `http-proxy` |
| CI/CD | GitHub Actions, Docker Hub |

## Repository structure

```text
Distributed-Lovable/
├── account-service/          # Authentication, accounts, and usage
├── api-gateway/              # External API gateway
├── common-lib/               # Shared security and common components
├── config-service/           # Centralized configuration
├── discovery-service/        # Service discovery
├── intelligence-service/     # AI generation and processing
├── workspace-service/        # Projects, files, and preview lifecycle
├── k8s/
│   ├── infra/                # Namespaces, ingress, policies, runner pool
│   ├── proxy/                # Wildcard preview proxy
│   ├── services/             # Application service deployments
│   └── stateful/             # PostgreSQL, Kafka, Redis, and MinIO
└── .github/workflows/        # Build and deployment workflows
```

## Prerequisites

Before running or deploying PromptForge, install:

- Java 21
- Maven, or use the included Maven wrappers
- Docker
- Kubernetes CLI (`kubectl`)
- Google Cloud CLI for GKE deployments
- Access to PostgreSQL, Kafka, Redis, and MinIO
- Required AI provider credentials

## Build a service locally

Each service contains its own Maven wrapper.

```bash
cd common-lib
./mvnw clean install -DskipTests

cd ../workspace-service
./mvnw clean package
```

Use the same process for the other Spring Boot services.

## Kubernetes deployment

Confirm that `kubectl` points to the intended cluster:

```bash
kubectl config current-context
kubectl get namespaces
```

Apply the infrastructure and service manifests as required:

```bash
kubectl apply -f k8s/infra/
kubectl apply -f k8s/stateful/
kubectl apply -f k8s/services/
kubectl apply -f k8s/proxy/proxy-deployment.yaml
```

Check the deployments:

```bash
kubectl get pods -n lovable-core
kubectl get pods -n lovable-previews -L status,project-id
```

## Preview operations

Check available and active runners:

```bash
kubectl get pods -n lovable-previews -L status,project-id
```

Follow workspace preview logs:

```bash
kubectl logs deployment/workspace-service \
  -n lovable-core \
  --follow
```

The expected baseline is two idle runner pods plus one busy pod for every active preview.

## Configuration

Runtime configuration is supplied through Kubernetes ConfigMaps, Secrets, and Spring Cloud Config.

Never commit credentials or local environment files. Sensitive values should be stored as Kubernetes Secrets or GitHub Actions secrets.

Examples include:

- Database passwords
- JWT signing keys
- MinIO credentials
- Docker registry credentials
- Google Cloud service-account configuration
- AI provider API keys

## Security

PromptForge includes:

- JWT authentication
- Service-level authorization
- Kubernetes RBAC
- Dedicated service accounts
- Namespace isolation
- Preview network policies
- TLS-enabled ingress
- Secret-based credential management
- Resource limits for preview containers

Preview environments should be treated as untrusted workloads and restricted from accessing internal services except where explicitly required.

## Cost controls

The preview platform is designed with several cost safeguards:

- A fixed pool of two idle runners
- CPU and memory limits
- Automatic expiry of preview pods
- Redis route expiration
- Namespace-level visibility
- Configurable preview lifetime

For production environments, configure Google Cloud budget alerts and regularly review GKE, networking, logging, and monitoring costs.

## Project status

PromptForge is under active development. APIs, deployment manifests, and preview behavior may change as the platform evolves.

## Roadmap

- Faster dependency installation and shared package caching
- Inactivity-based preview expiration
- Preview usage dashboards
- Project publishing workflows
- Improved generation history and rollback
- Team collaboration
- Additional AI model integrations
- Automated integration and end-to-end tests

## Contributing

Contributions, issues, and suggestions are welcome.

1. Fork the repository.
2. Create a feature branch.
3. Make and test your changes.
4. Commit with a clear message.
5. Open a pull request.

## Author

Built by [Deepak Jaiswar](https://github.com/DeepakJaiswar25).
