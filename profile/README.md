# Scalable Architecture Deployment on Kubernetes with a GitOps Approach in AWS using GitHub Actions and ArgoCD

## Team Members

- Luisa Castaño
- Juan Yustes
- Santiago Barraza

## Project Summary

This project implements an e-commerce solution based on a microservices architecture deployed on a Kubernetes cluster in AWS, applying advanced DevOps and GitOps practices to ensure end-to-end automation, scalability, security, and observability.

### Key System Components

- Microservices developed in Spring Boot, organized in independent repositories, packaged as Docker containers, and versioned in AWS ECR.
- Frontend built with Angular, also containerized and deployed in the cluster.
- Infrastructure as Code (IaC) implemented with Terraform, using S3 for remote state storage and DynamoDB for state locking.
- Orchestration using Kubernetes (EKS) with separate namespaces per environment (dev, stage, master) aligned with GitFlow branches.

### CI/CD Automation and GitOps

- GitHub Actions runs pipelines to build, test, build Docker images, scan for vulnerabilities (Trivy), and push them to ECR.
- Automatic update of image references in Helm Charts.
- ArgoCD watches the configuration repositories and syncs the cluster with the desired state defined in Git, eliminating manual deployment steps.

### Observability and Security

- Prometheus and Grafana for metrics monitoring.
- Elastic Stack (ELK) for centralized log management.
- Zipkin for distributed tracing.
- NetworkPolicies with Calico and RBAC following the principle of least privilege to segment and control traffic between services.
- Automated TLS certificates with Cert-Manager and Let's Encrypt, ensuring HTTPS for public endpoints.

### Implemented DevOps Best Practices

- Robust testing strategy: unit, integration, and end-to-end tests for each microservice.
- External Configuration Store pattern with centralized ConfigMaps for configuration management.
- Automatic secrets rotation implemented with CronJobs.
- Use of Persistent Volumes to ensure data persistence for databases and monitoring components.

## Added Value

This project practically demonstrates the integration of modern technologies and practices to design, deploy, and operate a distributed, scalable, and secure cloud platform, following industry standards in Kubernetes, AWS, GitOps, and CI/CD.

## Deployment and Operation of the Microservices Application

This guide details the steps needed to deploy and operate the microservices application in an AWS environment using GitHub Actions for CI/CD automation and ArgoCD for continuous deployment.

### Prerequisites

1. **AWS Account or AWS Lab:** You need access to an AWS environment. This project was developed and tested using an AWS Lab (such as AWS Learner Lab).
2. **AWS CLI Credentials:** When logging into the AWS account or lab, obtain temporary CLI credentials:
   * `AWS_ACCESS_KEY_ID`
   * `AWS_SECRET_ACCESS_KEY`
   * `AWS_SESSION_TOKEN`
3. **Project Repositories:** Make sure you have access to the following repositories:
   * `ecommerce-microservice-backend-app` (Backend)
   * `ecommerce-frontend-web-app` (Frontend)
   * `ecommerce-chart` (Helm Charts for ArgoCD)
   * `infrastructure` (Terraform code for AWS infrastructure)

### Initial Configuration of GitHub Secrets

The obtained AWS credentials must be set as **Secrets** in each of the GitHub repositories mentioned above.

1. Go to each repository on GitHub.
2. Navigate to `Settings` > `Secrets and variables` > `Actions`.
3. Create the following secrets with the values from your AWS credentials:
   * `AWS_ACCESS_KEY_ID`
   * `AWS_SECRET_ACCESS_KEY`
   * `AWS_SESSION_TOKEN`
   * Additionally, other secrets such as `TF_REGION`, `TF_ECR_NAME`, and `CHARTS_REPO_TOKEN` (a GitHub Personal Access Token for communication between pipelines) may be required according to each pipeline’s configuration. Check each pipeline for the full list of required secrets.

### Step 1: Deploy the Base Infrastructure on AWS

The infrastructure (VPC, EKS, ECR, etc.) is managed using Terraform.

1. **Repository:** `infrastructure`
2. **Pipeline:** Run the `deploy-infra.yaml` pipeline.
   * This pipeline applies the Terraform configuration to create or update the required infrastructure in AWS.
   * Wait for the pipeline to complete successfully. This ensures your Kubernetes cluster (EKS), container registry (ECR), and other resources are ready.

### Step 2: Initial Deployment of Application Images

Once the infrastructure is ready, you must build and push your application Docker images to ECR.

1. **Backend Microservices Deployment:**
   * **Repository:** `ecommerce-microservice-backend-app`
   * **Pipeline:** Run the main deployment pipeline (`deploy-and-trigger.yml`).
   * **Branches and Namespaces:**
     * **For a complete initial deployment to all environments/namespaces (dev, stage, master):** Manually run the pipeline for each environment branch you have configured (e.g., `dev`, `stage`, `master`). Each run will build the images, tag them properly, and update the Helm chart for the corresponding environment.
     * **For a deployment to `master` (production) only:** Run the pipeline selecting the `master` branch.

2. **Frontend Application Deployment:**
   * **Repository:** `ecommerce-frontend-web-app`
   * **Pipeline:** Run the main deployment pipeline of this repository.
   * **Branches and Namespaces:** Use the same logic as for the backend. Run the pipeline for each desired environment branch.

**Note on Initial Deployment:** Running it "three times" is a strategy to initially populate different namespaces or environments if your workflow maps branches (`dev`, `stage`, `master`) to different Kubernetes configurations or namespaces. If you only use one main environment (e.g., `master`), one execution is enough.

Once the images are in ECR and the Helm charts have been updated through their pipelines, ArgoCD (which must be configured in your EKS cluster and pointing to the `ecommerce-chart` repository) will detect the chart changes and automatically deploy/update the applications in Kubernetes.

### Step 3: Continuous Development and Deployment Flow (New Features and Changes)

Once the application is running, follow this flow to introduce changes or new features:

1. **Feature Development:**
   * Create a new feature branch from the main development branch (e.g., `dev`).
   * Make the necessary code changes in the backend (`ecommerce-microservice-backend-app`) or frontend (`ecommerce-frontend-web-app`).

2. **Pull Request (PR) and Code Check:**
   * When the feature is ready, create a Pull Request (PR) from your feature branch to the integration branch (e.g., `dev`).
   * When you open the PR, a "check" or "validation" pipeline (configured to run on `pull_request`) will execute automatically. This pipeline may include steps like build, unit tests, static code analysis, etc.
   * If the check pipeline passes, it indicates the code is safe to merge.

3. **PR Merge and Automatic Deployment:**
   * Once the PR is reviewed and approved, merge the PR into the integration branch (e.g., `dev`).
   * On merge, the main deployment pipeline (the same used in Step 2, but triggered by a `push` to the `dev` branch) will run automatically.
     * It will build new Docker images with the changes.
     * It will push the images to ECR with new tags.
     * **It will trigger the pipeline in the `ecommerce-chart` repository.**

4. **Chart Update and ArgoCD Synchronization:**
   * The pipeline in the `ecommerce-chart` repository (triggered by the previous step) will update the `values-master.yaml` file (or the appropriate values file for the branch/environment) to point to the new image tags in ECR.
   * These changes will be committed and pushed automatically to the `ecommerce-chart` repository.
   * **ArgoCD**, which monitors the `ecommerce-chart` repository, will detect these changes.
   * ArgoCD will automatically apply the changes to the Kubernetes cluster, updating the deployments to use the new images.

5. **Promotion to Other Environments (Stage, Production):**
   * Promoting changes from `dev` to `stage`, and then from `stage` to `master` (production), typically involves creating Pull Requests between these branches.
   * Each merge to a higher environment branch (`stage`, `master`) will trigger its respective deployment pipeline, which in turn will update the chart and be deployed by ArgoCD to the corresponding environment.

This automated flow ensures that changes are integrated, tested, and deployed consistently and efficiently in the cloud.
