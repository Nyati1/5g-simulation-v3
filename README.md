
# OVERVIEW

One of the biggest challenges for Mobile Service Providers is loss of revenue due to outages caused by configuration drift, inconsistent parameters, and risky manual deployments.  

This project demonstrates how this problem can be addressed using a GitOps-based promotion workflow for a simulated 5G network core environment using Kargo and Argo CD running on Akuity platform. The solution models how configuration changes move through development, staging, and production environments in a controlled and auditable way eliminating configuration drift, manual deployments, risky scripts and enabling predictable deployments.

Note, the deployment is only for minimal services with minimal parameters for demonstration.

# Architecture & Setup

## Components

* Kubernetes (K3s)
* Argo CD for GitOps deployment and reconciliation
* Kargo for promotion orchestration
* GitHub as the single source of truth
* Kustomize overlays for environment-specific configuration

## Core Directories

* base/: Contains the foundational Kubernetes manifests (amf-deployment.yaml, configmap.yaml, upf-deployment.yaml and the base kustomization.yaml). This serves as the "source of truth" for all environments.

* env/: Environment-specific overlays using Kustomize:

- dev/, staging/, prod/: Each directory contains unique configurations and patches tailored for that specific environment.

- argocd-management/: Houses the ApplicationSet definitions used by ArgoCD to automatically bootstrap the 5G core services across the cluster.

- kargo-management/: Contains the Kargo orchestration resources (project.yaml, projectconfig.yaml, stages.yaml, warehouse.yaml) used to manage versioned Freight and promote changes through the delivery pipeline.


# Deployment Steps 

## Development

1. Engineer updates configuration in: env/dev/kustomization.yaml

2. Change is committed to GitHub 'main'.

3. Kargo Warehouse detects the commit and generates freight.

4. Dev stage auto-promotes the freight.

5. Argo CD syncs the dev environment automatically.

6. Tests are done and once approved the configs are pushed to staging.

---

## Staging

1. Freight is manually promoted from dev to staging.

2. Kargo copies: env/dev/kustomization.yaml > env/staging/kustomization.yaml

3. Kargo commits the promotion change back to Git.

4. Argo CD detects the updated staging overlay and deploys the new configuration.

5. UAT tests are done and once approved the configs are pushed to production environment

---

## Production

1. Freight is manually promoted from staging to production.

2. Kargo copies: env/staging/kustomization.yaml > env/prod/kustomization.yaml

3. Kargo commits the updated production overlay to Git.

4. Argo CD deploys the promoted production configuration.

---

# Key Design Decisions

* Folder-Based Environment Model: The project uses a single 'main' branch with separate environment overlays (env/dev, env/staging and env/prod). Benefits include: easier repository structure, clear promotion visibility, simpler Git History, Easier demonstration of config changes 

* Namespace Isolation: Each environment has been deployed on it's own namespace. This is critical for 5G simulations because it allows running of multiple instances of the 5G core nodes on the same cluster without them interfering with each other's service discovery, networking or IP space.

* Automated Sync: By setting prune: true and selfHeal: true, the system is more robust e.g. if someone manually deletes a pod, Argo CD will detect the deviation from Git and immediately recreate it.

* The architecture adopts a microservices-based approach by decoupling the 5G Core into discrete functional units (AMF and UPF), enabling independent scaling and lifecycle management of the control and user planes. This design ensures that each network function can be updated or promoted through the GitOps pipeline without impacting the availability of the broader system. Although AMF and UPF are telecom network functions, in this project they are treated the same way modern applications treat microservices — independently deployed, independently configured, and promoted through environments using GitOps workflows.

*Manual promotion for staging and prod environments to ensure testing and approval are completed

### Tradeoff

This project uses a Kustomize-based base/ and env/ structure to prioritize configuration transparency over complex Helm templating. While this ensures  parameters are easy to audit during promotion, it trades off the scalability and efficiency provided by Helm's values.yaml files

---

## GitOps Deployment Model

Kargo does not deploy directly to Kubernetes but instead:

* Kargo updates Git
* Argo CD reconciles Git state into Kubernetes

This preserves Git as the single source of truth and provides a full audit trail for promotions.

---

# Assumptions

* GitHub is treated as the authoritative source of configuration truth.
* Environment configuration changes are promoted sequentially.
* Argo CD has continuous reconciliation enabled.
* Manual promotion approval is acceptable for staging and production.
* The environment is intended for demonstration purposes and not hardened for production 

# Future Improvements

* Deploy a complete 5G Network infrastructure for an end-end network demo
* Implement sync windows to ensure Argo CD does perform reconciliation outside of the maintenance window to avoid traffic disruptions.

# Example 1 Demo Scenario

1. Update MCC/MNC values in 'env/dev/kustomization.yaml'
2. Commit and push to GitHub
3. Observe freight generation in Kargo
4. Observe automatic deployment to dev
5. Promote freight to staging
6. Validate staging deployment
7. Promote freight to production
8. Observe production deployment via Argo CD

# Example 2 Demo Scenario

1. Update change the number of replicas under 'base/amf-deployment'
2. Commit and push to GitHub
3. Observe freight generation in Kargo
4. Observe automatic deployment to dev
5. Promote freight to staging
6. Validate staging deployment
7. Promote freight to production
8. Observe production deployment via Argo CD

###################################################################################
