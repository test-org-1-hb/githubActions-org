# Self-Hosted GitHub Actions Runners on Azure Kubernetes Service (AKS) with ARC

This guide provides a comprehensive walkthrough for setting up scalable, self-hosted GitHub Actions runners on Azure Kubernetes Service (AKS) using the [Actions Runner Controller (ARC)](https://github.com/actions/actions-runner-controller).

The setup includes:
- An **AKS cluster** with autoscaling enabled.
- A dedicated node pool for the controller and another for the runner pods.
- The **Actions Runner Controller (ARC)** to manage the lifecycle of runner pods.
- A **Runner Scale Set** configuration that uses **Docker-in-Docker (DinD)**, allowing your CI/CD jobs to build and run Docker containers.

## Prerequisites

Before you begin, ensure you have the following installed and configured:

1.  **Azure CLI**: [Installation Guide](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
2.  **kubectl**: [Installation Guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
3.  **Helm**: [Installation Guide](https://helm.sh/docs/intro/install/)
4.  An **Azure Subscription** with permissions to create resource groups, service principals, and AKS clusters.
5.  A **GitHub Personal Access Token (PAT)** with the appropriate scopes. See [Step 3](#step-3-create-a-github-personal-access-token-pat) for details.

---

## Step 1: Create the AKS Cluster

First, we'll create a foundational AKS cluster. This initial node pool (`nodepool1`) will be used to host the Actions Runner Controller itself.

```bash
# Define variables (replace with your values)
export RESOURCE_GROUP="MyResourceGroup"
export CLUSTER_NAME="MyAKSCluster"
export LOCATION="eastus" # Choose a region that suits you



# Create the AKS Cluster with an initial node pool for system services
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --node-count 1 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 3 \
  --node-vm-size Standard_D2s_v3 \
  --generate-ssh-keys

# Get the credentials to connect kubectl to your new cluster
az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME
```

---

## Step 2: Install the Actions Runner Controller (ARC)

The controller is the brain of the operation. It watches for workflow job events from GitHub and scales the runner pods up or down as needed. We will install it in its own namespace (`arc-systems`) and configure it to run on the initial node pool.

1.  **Create the `controllerValue.yml` file:**
    This file contains the configuration for the ARC controller. We use a `nodeSelector` to ensure the controller pods are scheduled on our initial node pool, which has the default label `agentpool: nodepool1`.

    ```yaml
    # controllerValue.yml

    # Enables High Availability (HA) with an active/passive model using leader election.
    replicaCount: 2

    # CRITICAL FOR PRODUCTION: Ensures pod stability and predictable performance.
    # Setting requests equal to limits gives the pod a "Guaranteed" Quality of Service (QoS).
    # You should monitor the controller's usage and adjust these values for your workload.
    # resources:
    #   limits:
    #     cpu: "100m"      # 0.5 CPU Core
    #     memory: "512Mi"  # 512 Mebibytes
    #   requests:
    #     cpu: "500m"
    #     memory: "512Mi"

    # Ensures the operator pods are scheduled ONLY on the initial node pool.
    # Verify this label on your nodes if you encounter scheduling issues.
    nodeSelector:
      agentpool: nodepool1

    # Recommended for production: Reduces the amount of log noise.
    flags:
      logLevel: "info"
    ```

2.  **Install the controller using Helm:**

    ```bash
    NAMESPACE="arc-systems"

    helm install arc \
      --namespace "${NAMESPACE}" \
      --create-namespace \
      -f controllerValue.yml \
      oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
    ```

---

## Step 3: Create a GitHub Personal Access Token (PAT)

ARC needs to authenticate to the GitHub API to register runners. Create a **classic** Personal Access Token with the correct scopes.

> **Security Warning**: Treat your PAT like a password. Do not hardcode it in public repositories. Use a secret management tool like Azure Key Vault for production environments.

1.  Navigate to your GitHub **Developer Settings** > [**Personal access tokens (classic)**](https://github.com/settings/tokens?type=beta).
2.  Generate a new token with the following scopes:

    * **For Repository-level Runners**: Select the `repo` scope.
    * **For Organization-level Runners**: Select the `admin:org` scope.

3.  Copy the generated token and save it securely. You will need it in creating kubernetes secret.
4.  create kubernetes secrate
```bash
GITHUB_PAT=ghp_IK9m...........

kubectl create secret generic arc-github-secret \
  --namespace=arc-runners \
  --from-literal=github_token='${GITHUB_PAT}'
```

---


## Step 4: Add a Dedicated Node Pool for Runners

It's a best practice to run your GitHub Actions jobs on a separate node pool. This isolates workloads, allows for different VM sizes, and prevents CI jobs from impacting cluster-critical services.

We will use a **node taint** (`runners-only=true:NoSchedule`) to ensure that only our runner pods can be scheduled on this pool.

```bash
# Use the variables defined in Step 1
# RESOURCE_GROUP="MyResourceGroup"
# CLUSTER_NAME="MyAKSCluster"

az aks nodepool add \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --name runnerpool \
  --node-count 1 \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 5 \
  --node-taints "runners-only=true:NoSchedule"
```
* `--min-count 0`: Allows the node pool to scale down to zero nodes when there are no active jobs, saving costs.
* `--node-taints`: Prevents other pods from being scheduled here unless they have a matching "toleration."

---

## Step 5: Deploy the Runner Scale Set

Finally, we deploy the `RunnerScaleSet`. This custom resource defines the template for the runner pods that will execute your workflows.

1.  **Create the `runner-values.yml` file:**
    This configuration defines everything about our runners, including the GitHub repository/organization, resource requests, and the crucial Docker-in-Docker setup.

    > **IMPORTANT**: You must update the placeholder values (`<...>`) in the file below.

    ```yaml
    # runner-values.yml

    runnerScaleSetName: "aks-dind-runner-set" 
    namespace: "arc-runners"
    githubConfigUrl: "(https://github.com/)<your-org>/<your-repo>" # <-- CHANGE THIS
    
    # !! SECURITY WARNING !!
    # Do not commit your PAT to version control.
    # For production, use a pre-existing Kubernetes secret and reference it via `githubConfigSecret.name`.
    githubConfigSecret: "arc-github-secret"  # <-- CHANGE THIS FROM THE NAME GIVEN IN STEP 3

    # The service account of the controller installed in Step 2
    controllerServiceAccount:
      name: arc-gha-rs-controller
      namespace: arc-systems

    # We disable the chart's automatic container mode to define our own sidecars.
    containerMode:
      type: ""

    template:
      spec:
        # Schedule pods on our dedicated runner node pool
        nodeSelector:
          agentpool: runnerpool
        # Tolerate the taint applied to the runner node pool
        tolerations:
        - key: "runners-only"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"

        volumes:
          - name: work
            emptyDir: {}
          - name: dind-sock
            emptyDir: {}

        # We explicitly define two containers: the runner and a Docker-in-Docker (dind) sidecar.
        containers:
          # 1. The main runner container
          - name: "runner"
            image: "ghcr.io/actions/actions-runner:latest"
            command: ["/home/runner/run.sh"]
            resources:
              requests:
                cpu: "100m"
                memory: "256Mi"
              limits:
                cpu: "500m"
                memory: "512Mi"
            env:
              - name: DOCKER_HOST
                value: unix:///var/run/docker.sock
            volumeMounts:
              - name: work
                mountPath: /home/runner/_work
              - name: dind-sock
                mountPath: /var/run

          # 2. The Docker-in-Docker sidecar container
          - name: "dind"
            image: "docker:dind"
            securityContext:
              privileged: true
            args:
              - dockerd
              - --host=unix:///var/run/docker.sock
            volumeMounts:
              - name: work
                mountPath: /home/runner/_work
              - name: dind-sock
                mountPath: /var/run

    # Optional: Disable metrics scraping if you don't use Prometheus
    metrics:
      enabled: false
    ```

2.  **Install the runner set using Helm:**

    ```bash
    helm install "aks-dind-runner-set" \
      --namespace "arc-runners" \
      --create-namespace \
      -f runner-values.yml \
      oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
    ```

---

## Verification and Usage

1.  **Check the Controller Pods:**
    ```bash
    kubectl get pods -n arc-systems
    ```
    You should see two `arc-gha-rs-controller-...` pods running.

2.  **Check the Runner Pods:**
    Initially, you won't see any runner pods.
    ```bash
    kubectl get pods -n arc-runners
    ```
    Trigger a GitHub Actions workflow in your configured repository.


You are all set! Your AKS cluster will now automatically provide ephemeral, container-capable runners for your GitHub Actions workflows.
