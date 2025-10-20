# Argo CD Deployment  
> GitOps style infrastructure & application deployment using Argo CD on Kubernetes

## Table of Contents  
1. [Project Overview](#project-overview)  
2. [Why Use Argo CD & GitOps](#why-use-argocd-gitops)  
3. [Repository Structure](#repository-structure)  
4. [Getting Started](#getting-started)  
   - [Prerequisites](#prerequisites)  
   - [Installation & Setup](#installation-setup)  
   - [Deploying to a Cluster](#deploying-to-a-cluster)  
5. [Folder & File Breakdown](#folder-file-breakdown)  
6. [Usage](#usage)  
   - [Adding a New Application](#adding-a-new-application)  
   - [Syncing/Updating](#syncing-updating)  
7. [Best Practices & Patterns](#best-practices-patterns)  
8. [Troubleshooting & FAQs](#troubleshooting-faqs)  
9. [Contributing](#contributing)  
10. [License](#license)  
11. [Authors & Acknowledgements](#authors-acknowledgements)  
12. [Contact](#contact)  

---

## Project Overview  
This repository defines a GitOps-centric deployment architecture using Argo CD, enabling declarative, version-controlled, auditable application and infrastructure deployments on Kubernetes.  
With Argo CD, you push your application manifests (Helm charts, Kustomize overlays, plain YAML) into Git, and Argo continuously ensures your cluster state matches the declared state.  
This repo serves as the central source of truth for one or more environments (e.g., `dev`, `staging`, `prod`) and streamlines how changes flow from dev to production.

---

## Why Use Argo CD & GitOps  
Using Argo CD and GitOps brings numerous advantages:

- **Declarative control**: Application configuration is version controlled in Git.  
- **Auditable changes**: All changes are traceable through Git history.  
- **Automated delivery**: Argo monitors Git and reconciles changes automatically.  
- **Reduced drift**: Ensures the live cluster matches the declared state.  
- **Environment promotion**: Easily manage multiple environments, promoting changes from `dev → staging → prod`.  
- **Rollback simplicity**: Revert Git commit → cluster state reverts.  

These benefits align with modern infrastructure delivery and help reduce manual errors, increase transparency, and improve deployment velocity. :contentReference[oaicite:1]{index=1}

---

## Repository Structure  
Below is a typical high-level view of how the repository is organized:
```
argocd-deployment/
├── argo-cd/ # manifests for Argo CD installation (namespace, CRDs, RBAC)
│ ├── base/
│ └── overlays/
├── apps/
│ ├── dev/ # applications for dev environment
│ ├── staging/ # applications for staging environment
│ └── prod/ # applications for production environment
├── charts/ # (optional) Helm charts managed here
├── infrastructure/ # infra resources (namespaces, ingress, config-maps)
│ ├── base/
│ └── overlays/
├── docs/ # documentation, architecture diagrams
├── .github/
│ └── workflows/ # GitHub Actions or CI/CD pipelines
├── helmfile.yaml # if using Helmfile to manage multiple charts
├── kustomization.yaml # if using Kustomize
├── README.md # this file
└── LICENSE

```

> **Note:** Actual folder names and structure may vary. Adjust the above as needed.

---

## Getting Started

### Prerequisites  
Before you begin, ensure you have the following:  
- A Kubernetes cluster (e.g., kind, GKE, EKS, AKS, self-hosted)  
- `kubectl` CLI configured and pointed at the target cluster  
- `argocd` CLI optionally installed (for command-line operations)  
- Git repository access and permissions  
- (Optional) Helm or Kustomize installed if using those tools  
- (Optional) GitHub Actions or another CI pipeline if you automate changes  

### Installation & Setup  
1. **Clone the repository**  
   ```
   git clone https://github.com/majevince/argocd-deployment.git
   cd argocd-deployment
   ```
   
2. Install Argo CD (if not already installed)
Using manifests in argo-cd/base / overlays:

```
kubectl apply -k argo-cd/overlays/cluster-install
```

3. Configure Argo admin access
Retrieve initial password, log in via CLI or UI, and change default credentials.

4. Connect your Git repository
In Argo UI or via CLI: register this repo as a Git source so Argo can monitor it.

### Deploying to a Cluster
Once setup is complete:

Navigate to the folder apps/dev/your-app/ (or whichever environment)

Ensure your Application manifest or Helm/Kustomize config is correct

Commit your changes and push to Git

Argo picks up changes and syncs automatically (or manually trigger)

### Folder & File Breakdown
Here’s more detailed explanation of each major component:

argo-cd/
This folder contains manifests and configuration to install and configure Argo CD itself (namespace, CRDs, roles, service accounts, ingress, autologin config).

base/: vanilla install

overlays/cluster-name/: environment specific tweaks (ingress, OIDC config, resource limits)

apps/
This is where your actual applications live. Organized by environment:

dev/

staging/

prod/
Within each environment folder you’ll find one folder per application with the required K8s manifests or Helm/Kustomize files.
Then you may have a top-level Application.yaml manifest referencing that folder for Argo to sync.

infrastructure/
Infrastructure resources that are shared across applications or environments (namespaces, ingress controllers, storage classes, config maps, secrets defined via sealed secrets or SOPS).
Use base/ and overlays/ for environment variation.

charts/
(Optional) If you manage custom Helm charts for your applications, store them here. Argo can reference charts from this folder or external helm repos.

.github/workflows/
CI / CD workflows such as:

Linting manifests

Validating K8s syntax (kube-val, helm lint)

Testing environment deployments (kind cluster)

Creating or updating Application manifests when new apps are added

Triggering Argo sync via CLI or webhook

Other files
helmfile.yaml or Kustomization.yaml if you use Helmfile or Kustomize to manage templated/parameterised deployments

README.md — this file

LICENSE — open source license

## Usage
### Adding a New Application
1. Create a new folder under apps/<environment>/<app-name>/

2. Add Kubernetes manifests (Deployment, Service, Ingress/Route) or a Helm chart reference

3. Create an Argo CD Application manifest referencing this folder or chart. Example:

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/majevince/argocd-deployment.git
    targetRevision: HEAD
    path: apps/dev/my-app
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app-namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

```
4. Commit & push the changes → Argo CD will detect the new Application and start syncing it.

### Syncing / Updating
- Manual sync via Argo UI or CLI:

```
argocd app sync <application-name>
```
- Automatic sync if selfHeal and automated are enabled: Argo continuously watches Git and applies changes.

- Rollback: revert the Git commit → Argo relays the change and rolls back the cluster state.

## Best Practices & Patterns
- Keep env-specific values in overlays, avoid hard-coding secrets.

- Use immutable tags (e.g., image tags) not latest so you can trace what was deployed.

- Use syncPolicy.prune = true and selfHeal = true to clean up deleted resources and self-correct drift.

- Branch strategy: e.g., main for prod, staging for staging, dev for dev. Merge flows with PRs.

- Leverage ApplicationSets if you manage many similar apps or multi-cluster setups.

- Use health checks & status monitoring in Argo to track application health.

- Keep manifests small and focused; break large apps into microservices managed individually.

Secure Argo CD: enable RBAC, SSO/OIDC, TLS for the UI, restrict cluster admin access.

## Troubleshooting & FAQs
### Q: Argo shows “OutOfSync” but no changes in Git

- Check Git repo path and targetRevision are correct.

- Check file permissions or Git branch updates.

- Ensure argocd.repoCache is updated (sometimes delay).

### Q: Application stuck in “Sync Failed”

- Use kubectl describe or check Argo UI Events to find resource errors (e.g., invalid spec).

- Helm/Kustomize templates may have missing values; validate locally first.

### Q: How to roll back a deployment?

- In Git: revert the commit which introduced the change → push. Argo will notice and revert live state.

- In Argo UI: pick a previous revision and click “Rollback”.

### Q: Can I deploy Argo CD itself via Argo CD?
Yes — you can declare Argo CD’s own manifests in this repo and let Argo manage itself (a “self-bootstrap” pattern). 
GitHub
+1

Contributing
Contributions are welcome! Here’s how you can help:

1. Fork the repository.

2. Create a feature branch: git checkout -b feature/my-new-feature

3. Run any validation/lint checks locally (e.g., kube-val, helm lint).

4. Commit with clear, descriptive messages.

5. Push your branch and open a Pull Request.

6. Ensure that CI passes and reviewers approve changes.

Please follow the Contributor Code of Conduct in this project.

### License
This project is licensed under the MIT License — see the LICENSE file for details.

Authors & Acknowledgements
Vincent Anyah (majevince) — initial setup and architecture.
Email lptxanyahvincent@gmail.com

Thanks to the community and the maintainers of Argo CD for providing the GitOps tooling that powers modern Kubernetes deployments.

Inspired by example repositories such as argocd‑example‑apps which demonstrate GitOps patterns. 
GitHub

## Contact
For questions or issues, please open an issue in this repository or reach out at lptxanyahvincent@gmail.com.

“Declare it once in Git. Argo CD will ensure the cluster lives it.”

Thank you for using this GitOps infrastructure!

---

✅ **Next Steps**  
- Replace placeholder fields like *Your Name* and *lptxanyahvincent@gmail.com* with real values.  
- Update any version numbers, Kubernetes cluster details, or environment names to reflect your real setup.  
- If there are special workflows (e.g., multi-cluster, ApplicationSets, helm-chart publishing) you can add additional sections.  
- Consider adding badges at the top (build status, license, coverage) for visual polish.

Would you like me to generate **badge markup**, or perhaps **automatically extract the folder structure** of your repo and include it in the README?
::contentReference[oaicite:5]{index=5}
