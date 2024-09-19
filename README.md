# FITA GitOps Repository

Welcome to the `fta_gitops` repository! This repository manages the GitOps workflow for deploying Helm charts related to the `fta-users` application. Below, you'll find a detailed overview of our CI/CD processes and Helm chart configurations.

## Table of Contents

- [CI/CD GitOps](#cicd-gitops)
  - [Pipeline Overview](#pipeline-overview)
  - [Updating Helm Charts with New Image Tags](#updating-helm-charts-with-new-image-tags)
  - [Automated Deployment with GitHub Actions and ArgoCD](#automated-deployment-with-github-actions-and-argocd)
- [Helm Charts](#helm-charts)
  - [Helm Charts Overview](#helm-charts-overview)
  - [Managing Secrets with ArgoCD Vault Plugin](#managing-secrets-with-argocd-vault-plugin)
  - [Mounting Kubernetes Secrets as Environment Files](#mounting-kubernetes-secrets-as-environment-files)
  - [Workload Identity with Service Accounts](#workload-identity-with-service-accounts)
  - [Automated DNS Records, SSL Certificates, and Ingress](#automated-dns-records-ssl-certificates-and-ingress)
  - [Horizontal Pod Autoscaling (HPA) and Service Configuration](#horizontal-pod-autoscaling-hpa-and-service-configuration)

---

## CI/CD GitOps

### Pipeline Overview

Our CI/CD pipeline is designed to automate the build, tagging, and deployment of Docker images for the `fta-users` application. The workflow is triggered on pushes to the `main` and `dev` branches, as well as on tags matching the pattern `v*`. Here are the key stages of the pipeline:

1. **Checkout Code**: The pipeline starts by checking out the latest code from the repository.
2. **Set Up Docker Buildx**: Prepares the environment for building Docker images.
3. **Login to Docker Hub**: Authenticates with Docker Hub using stored secrets.
4. **Set Image Tag**: Determines the appropriate image tags based on the branch or tag being pushed.
5. **Build, Tag, and Push**: Builds the Docker image, tags it, and pushes it to the Docker repository.
6. **Retag and Push for Release**: Handles tagging for release versions.
7. **Update Helm Chart**: Updates the Helm chart with the new image tag and pushes the changes to the GitOps repository.

### Updating Helm Charts with New Image Tags

After successfully building, tagging, and pushing the Docker image, the pipeline proceeds to update the Helm charts with the new image tag. This involves:

- **Modifying `values.yaml`**: The `values.yaml` file within the Helm chart is updated to reference the new Docker image repository.
- **Updating `Chart.yaml`**: The `appVersion` field in `Chart.yaml` is set to the new image tag, ensuring that the Helm chart reflects the latest application version.

These updates ensure that the deployment configurations are always in sync with the latest Docker images.

### Automated Deployment with GitHub Actions and ArgoCD

Once the Helm charts are updated, the changes are pushed to the GitOps repository using GitHub Actions. ArgoCD monitors this repository and automatically synchronizes the application state with the Kubernetes cluster. The synchronization process involves:

1. **Pushing Updates**: GitHub Actions commits and pushes the updated Helm charts to the specified branch in the GitOps repository.
2. **ArgoCD Sync**: ArgoCD detects the changes and initiates a synchronization process to apply the updated configurations to the Kubernetes cluster, ensuring the application is deployed with the latest Docker image.

This seamless integration between GitHub Actions and ArgoCD facilitates continuous deployment with minimal manual intervention.

---

## Helm Charts

### Helm Charts Overview

Our Helm charts are structured to support three distinct environments:

- **Incubator**: Used for development purposes.
- **Test**: Serves as the staging environment.
- **Stable**: Dedicated to production deployments.

This separation ensures that each environment can be managed independently, allowing for controlled testing and deployment processes.

### Managing Secrets with ArgoCD Vault Plugin

To handle sensitive information securely, we utilize the ArgoCD Vault Plugin. Here's how secrets are managed:

- **Secret Placeholders**: Within the `values.yaml` file, secrets are defined using placeholders. For example:

  ```yaml
  appSecret:
    secrets:
      username: <fta-dev-svc-users | jsonPath {.USERNAME}>
      password: <fta-dev-svc-users | jsonPath {.PASSWORD}>
      # Additional secrets...
  ```

- **Vault Replacement**: During deployment, the ArgoCD Vault Plugin replaces these placeholders with actual values retrieved from the Google Secret Manager. This approach ensures that sensitive data is not stored directly in the repository.

### Mounting Kubernetes Secrets as Environment Files

To provide the application with necessary configuration parameters, Kubernetes secrets are mounted as environment files within the pods. This is achieved by:

- **Defining a Secret**: The `secret.yaml` file creates a Kubernetes Secret containing the environment variables.

  ```yaml
  stringData:
    .env: |-
      username='actual_username'
      password='actual_password'
      # Additional variables...
  ```

- **Volume Mounting**: In the `deployment.yaml`, the secret is mounted as a file at `/app/.env` inside the container.

  ```yaml
  volumeMounts:
    - name: env-secret
      mountPath: "/app/.env"
      subPath: .env
      readOnly: true
  ```

- **Usage by Application**: The `fta-users` application reads the `.env` file during runtime to access the necessary environment variables.

### Workload Identity with Service Accounts

We employ Workload Identity to enhance security and manage permissions effectively:

- **Service Account Configuration**: The `serviceaccount.yaml` defines a Kubernetes Service Account with annotations linking it to a Google Cloud service account.

  ```yaml
  annotations:
    iam.gke.io/gcp-service-account: "fta-svc-users@fta-platform.iam.gserviceaccount.com"
  ```

- **Workload Identity Integration**: This setup allows the Kubernetes pods to assume the identity of the specified Google Cloud service account, enabling secure access to Google Cloud resources without managing long-lived credentials.

### Automated DNS Records, SSL Certificates, and Ingress

Our Helm charts automate several critical aspects of application networking and security:

- **External DNS Records**: Annotations in `ingress.yaml` enable the creation of DNS records automatically.

  ```yaml
  annotations:
    external-dns.alpha.kubernetes.io/hostname: "users.dev.fta.blast.co.id"
    external-dns.alpha.kubernetes.io/ttl: "300"
  ```

- **SSL Certificates with Cert-Manager**: The `cert-manager` handles the issuance and management of SSL certificates.

  ```yaml
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-fta"
  ```

- **Ingress Configuration**: Using NGINX Ingress, the `ingress.yaml` defines routing rules and TLS settings to secure HTTP traffic.

  ```yaml
  spec:
    ingressClassName: "nginx"
    tls:
      - secretName: users-tls
        hosts:
          - users.dev.fta.blast.co.id
    rules:
      - host: users.dev.fta.blast.co.id
        http:
          paths:
            - path: /
              pathType: ImplementationSpecific
              backend:
                service:
                  name: fta-users
                  port:
                    number: 3000
  ```

This automation ensures that networking and security configurations are consistently applied across environments.

### Horizontal Pod Autoscaling (HPA) and Service Configuration

To ensure application scalability and reliability, we implement Horizontal Pod Autoscaling and define service exposure:

- **Horizontal Pod Autoscaler (HPA)**: The `hpa.yaml` configures HPA to automatically adjust the number of pod replicas based on CPU and memory utilization.

  ```yaml
  spec:
    minReplicas: 1
    maxReplicas: 4
    metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 75
      - type: Resource
        resource:
          name: memory
          target:
            type: Utilization
            averageUtilization: 75
  ```

- **Service Exposure**: The `service.yaml` defines a Kubernetes Service that exposes the application on port `3000`.

  ```yaml
  spec:
    type: ClusterIP
    ports:
      - port: 3000
        targetPort: http
        protocol: TCP
        name: http
    selector:
      app: fta-users
  ```

These configurations ensure that the `fta-users` application can scale according to demand while being reliably accessible within the Kubernetes cluster.

---

## Getting Started

To get started with the `fta_gitops` repository:

1. **Clone the Repository**:

   ```bash
   git clone git@github.com:your-org/fta_gitops.git
   ```

2. **Configure GitHub Actions**: Ensure that the necessary secrets (e.g., `DOCKER_USERNAME`, `DOCKER_PASSWORD`, `GITOPS_SSH_PRIVATE_KEY`) are set in your GitHub repository settings.

3. **Deploy with ArgoCD**: Make sure ArgoCD is configured to monitor the `fta_gitops` repository and the appropriate branches for deployment.

4. **Customize Helm Charts**: Modify the Helm charts as needed for your specific environment or application requirements.

For detailed instructions, please refer to the [Contributing](CONTRIBUTING.md) and [Documentation](docs/) sections.

---

## Contributing

We welcome contributions! Please follow our [contribution guidelines](CONTRIBUTING.md) to submit issues or pull requests.

## License

This project is licensed under the [MIT License](LICENSE).

## Contact

For any inquiries or support, please contact [imam.arief.rhmn@gmail.com](mailto:imam.arief.rhmn@gmail.com).

---

Thank you for using the `fta_gitops` repository! We strive to maintain a robust and secure deployment pipeline to ensure the reliability and scalability of the `fta-users` application.