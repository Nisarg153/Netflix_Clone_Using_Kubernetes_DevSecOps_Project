# Netflix Clone — DevSecOps CI/CD Pipeline with Jenkins, Docker, Kubernetes & Security Scanning

## About This Project

This project deploys a Netflix clone application through a full DevSecOps CI/CD pipeline. The pipeline integrates security scanning at multiple stages — static code analysis with SonarQube, dependency vulnerability checks with OWASP Dependency-Check, and container image scanning with Trivy — before deploying to a Kubernetes cluster using ArgoCD.

I built this to get hands-on experience with shift-left security in a CI/CD pipeline, understanding how to catch vulnerabilities early in the development cycle rather than after deployment.

> **Inspired by:** [CloudChamp's DevSecOps project tutorial](https://www.youtube.com/@cloudchamp) — I followed the core project concept and then wrote my own Jenkinsfile, configured the Docker build, and created the Kubernetes manifests for my infrastructure setup.

---

## Architecture

```
Developer Push
      │
      ▼
┌──────────┐     ┌───────────────────────────────────────────┐
│  GitHub   │────▶│              Jenkins (EC2)                │
│  (Source) │     │                                           │
└──────────┘     │  ┌─────────┐  ┌────────┐  ┌───────────┐  │
                 │  │SonarQube│  │ OWASP  │  │  Trivy    │  │
                 │  │(Quality)│  │(Deps)  │  │(Container)│  │
                 │  └─────────┘  └────────┘  └───────────┘  │
                 │                                           │
                 │  ┌─────────────────────────────────────┐  │
                 │  │  Build Docker Image → Push to       │  │
                 │  │  DockerHub → Update K8s Manifests   │  │
                 │  └─────────────────────────────────────┘  │
                 └───────────────────┬───────────────────────┘
                                     │
                              Manifest Update
                                     │
                                     ▼
                             ┌───────────────┐
                             │    ArgoCD      │
                             │ (GitOps Sync)  │
                             └───────┬───────┘
                                     │
                                     ▼
                             ┌───────────────┐
                             │  Kubernetes    │
                             │  (Minikube)    │
                             └───────────────┘
                                     │
                              ┌──────┴──────┐
                              │  Prometheus  │
                              │  & Grafana   │
                              │ (Monitoring) │
                              └─────────────┘
```

---

## Pipeline Stages

| Stage | Tool | Purpose |
|-------|------|---------|
| **Checkout** | Git | Pulls latest code from GitHub |
| **SonarQube Analysis** | SonarQube | Static code analysis — detects bugs, code smells, and security hotspots |
| **Build & Test** | npm | Installs dependencies and builds the application |
| **OWASP Dependency Check** | OWASP DC | Scans project dependencies for known CVEs |
| **Trivy FS Scan** | Trivy | Scans the filesystem for vulnerabilities before containerization |
| **Build & Push Image** | Docker | Builds the container image with TMDB API key and pushes to DockerHub |
| **Update Manifests** | Shell/Git | Updates Kubernetes deployment YAML with the new image tag |
| **Deploy** | ArgoCD | Syncs the updated manifests to the Kubernetes cluster |

---

## Security Tools Breakdown

| Tool | What It Catches | When It Runs |
|------|----------------|--------------|
| **SonarQube** | Code bugs, vulnerabilities, code smells, security hotspots | After checkout, before build |
| **OWASP Dependency-Check** | Known CVEs in npm/Node.js dependencies (via NVD database) | After build |
| **Trivy** | OS-level and application vulnerabilities in the filesystem and Docker images | After build, before push |

This layered approach ensures vulnerabilities are caught at the code level, dependency level, and container level — before anything reaches production.

---

## Infrastructure Setup

| Component | Where It Runs |
|-----------|---------------|
| Jenkins Server | AWS EC2 (Ubuntu) |
| SonarQube | Docker container on EC2 |
| OWASP Dependency-Check | Jenkins plugin on EC2 |
| Trivy | Installed on EC2 |
| Docker Registry | DockerHub |
| Kubernetes Cluster | Minikube (local) |
| ArgoCD | Deployed on Minikube |
| Monitoring | Prometheus & Grafana on Minikube via Helm |

---

## Tech Stack

- **Application:** Netflix clone (React/TypeScript + TMDB API)
- **CI Server:** Jenkins (custom Jenkinsfile)
- **Security:** SonarQube, OWASP Dependency-Check, Trivy
- **Containerization:** Docker
- **Container Registry:** DockerHub
- **GitOps:** ArgoCD
- **Orchestration:** Kubernetes (Minikube)
- **Monitoring:** Prometheus & Grafana (Helm-based install)
- **API:** TMDB (The Movie Database) for movie data

---

## Project Structure

```
├── src/                          # React/TypeScript application source
├── public/                       # Static assets
├── Kubernetes/
│   └── deployment.yml            # K8s deployment & service manifests (customized)
├── Dockerfile                    # Docker build config with TMDB API key arg
├── pipeline.txt                  # Jenkins pipeline reference
├── package.json                  # Node.js dependencies
├── vite.config.ts                # Vite build configuration
└── README.md
```

---

## What I Built & Customized

- **Jenkinsfile:** Wrote the full pipeline from scratch — checkout, SonarQube integration, npm build, OWASP scan, Trivy scan, Docker build/push, and automated Git manifest updates with the new image tag.
- **Docker Configuration:** Configured the Dockerfile and build args to inject the TMDB API key at build time.
- **Kubernetes Manifests:** Created the deployment and service YAML files for deploying the Netflix clone on Minikube, including the `sed`-based image tag update in the pipeline.
- **End-to-End Deployment:** Got the full pipeline working — from a Git push triggering Jenkins, through all security scans, to the app running on Kubernetes via ArgoCD.

---

## How to Run This Project

### Prerequisites

- AWS EC2 instance (t2.large recommended — SonarQube needs RAM)
- Docker installed
- Minikube and kubectl installed
- Jenkins with Java 17
- TMDB API key ([get one here](https://www.themoviedb.org/settings/api))
- DockerHub account

### Steps

1. **Set up Jenkins on EC2** with required plugins:
   - SonarQube Scanner, NodeJS, OWASP Dependency-Check, Docker Pipeline

2. **Run SonarQube:**
   ```bash
   docker run -d --name sonarqube -p 9000:9000 sonarqube:lts-community
   ```

3. **Install Trivy on EC2:**
   ```bash
   sudo apt-get install wget apt-transport-https gnupg lsb-release
   wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
   echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
   sudo apt-get update && sudo apt-get install trivy
   ```

4. **Configure Jenkins Credentials:**
   - DockerHub credentials
   - SonarQube token
   - GitHub token (for manifest updates)

5. **Start Minikube and deploy ArgoCD:**
   ```bash
   minikube start --memory=4096
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

6. **Set up monitoring (optional):**
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo add grafana https://grafana.github.io/helm-charts
   helm install prometheus prometheus-community/prometheus -n monitoring --create-namespace
   helm install grafana grafana/grafana -n monitoring
   ```

7. **Create the Jenkins pipeline job** pointing to this repo and trigger a build.

8. **Access the application:**
   ```bash
   kubectl port-forward svc/netflix-app 30007:80
   ```
   Open http://localhost:30007

---

## Key Learnings

- **Shift-left security** is more than a buzzword — integrating SonarQube, OWASP, and Trivy at different pipeline stages caught issues that would have gone unnoticed in a build-only pipeline.
- **OWASP Dependency-Check** takes time on first run (downloads the NVD database), which is important to account for in pipeline timeout settings.
- **Jenkins + ArgoCD separation** keeps CI and CD cleanly decoupled — Jenkins doesn't need kubectl access to the cluster, it just updates a Git manifest and ArgoCD handles the rest.
- **Docker build args** (`--build-arg`) are the right way to inject API keys at build time without hardcoding them in the Dockerfile.

---

## Screenshots

> **Note:** Screenshots will be added soon.
>
> _Placeholder for:_
> - [ ] Jenkins pipeline with all stages passing
> - [ ] SonarQube analysis dashboard
> - [ ] OWASP Dependency-Check report
> - [ ] Trivy scan output
> - [ ] ArgoCD sync status
> - [ ] Netflix clone running on Kubernetes

---

## Future Improvements

- [ ] Add Slack notifications for pipeline failures and security scan alerts
- [ ] Implement SonarQube quality gates to fail the build on critical issues
- [ ] Add container image scanning with Trivy before DockerHub push (in addition to FS scan)
- [ ] Deploy to AWS EKS for a production-like environment
- [ ] Add Grafana dashboards for application-level metrics
- [ ] Implement Jenkins shared libraries for pipeline reusability
