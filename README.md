## Guide: Deploying GitHub Actions Runner Controller (ARC) on AKS 


-----

### Step 1: Connect to Your AKS Cluster & Verify Access

First, you need to configure `kubectl` to communicate with your AKS cluster.

1.  **Get Cluster Credentials**: Use the Azure CLI to fetch the credentials for your cluster. This command updates your local `kubeconfig` file.

    ```bash
    az aks get-credentials --resource-group <your-resource-group> --name <your-cluster-name>
    ```

2.  **Verify Context**: Confirm that `kubectl` is pointing to the correct cluster context.

    ```bash
    kubectl config get-contexts
    ```

    The current context will be marked with an asterisk (\*).

3.  **Inspect Cluster Resources (Optional)**: To get a high-level overview of everything running in your cluster (pods, services, deployments, etc.) across all namespaces, use:

    ```bash
    kubectl get all --all-namespaces
    ```

    For a more exhaustive list that includes objects like ConfigMaps, Secrets, and CRDs, you can run the following command:

    ```bash
    kubectl api-resources --verbs=list --namespaced -o name \
      | xargs -n 1 kubectl get --show-kind --ignore-not-found --all-namespaces
    ```

-----

### Step 2: Install the Actions Runner Controller (ARC)

The controller is the "brain" of ARC. It's a Kubernetes operator that watches for new jobs on GitHub and manages the lifecycle of runner pods.

  * **What is ARC?** Actions Runner Controller (ARC) is a Kubernetes operator that automates the deployment, scaling, and management of self-hosted GitHub Actions runners. It listens for queued workflow jobs and creates ephemeral runner pods on-demand.
  * **Key Components**:
      * **Operator (Manager Pod)**: The control plane that reconciles ARC resources with GitHub's job queue.
      * **Custom Resource Definitions (CRDs)**: Defines custom resources like `AutoScalingRunnerSet` that you will use to configure your runners.

<!-- end list -->

1.  **Install the Controller via Helm**: Use the following command to install the controller into its own namespace.

    ```bash
    NAMESPACE="arc-systems"
    helm install arc \
      --namespace "${NAMESPACE}" \
      --create-namespace \
      oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
    ```

2.  **Configuration for Different Environments**: The controller's behavior can be customized using a `values.yaml` file. Here are some recommended configurations for different environments.

      * **Development (Low Load: 1-5 Concurrent Jobs)**

          * **Goal**: Ensure stability on a shared cluster without consuming excessive resources.
          * **Action**: Create a `values.yaml` file with minimal resource requests and keep the replica count at 1.

        <!-- end list -->

        ```yaml
        # dev-values.yaml
        replicaCount: 1
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
        ```

      * **Production (High Load: 10-50+ Concurrent Jobs)**

          * **Goal**: Ensure reliability, high availability, and scalability for frequent or bursty job loads.
          * **Action**: Set `replicaCount` to 2 for high availability (HA) and provide more robust resource limits. Enable metrics for monitoring.

        <!-- end list -->

        ```yaml
        # prod-values.yaml
        replicaCount: 2
        metrics:
          enabled: true # For Prometheus monitoring
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 1
            memory: 1Gi
        ```

-----

### Step 3: Create a GitHub Personal Access Token (PAT)

ARC needs a **Personal Access Token (PAT)** to authenticate with the GitHub API, allowing it to manage runners for your repository or organization.

1.  **Generate a New Token**:

      * Go to your GitHub **Developer settings** \> **Personal access tokens** \> **Tokens (classic)**.
      * Click **Generate new token**.

2.  **Select Scopes**: The required permissions depend on where you are installing the runners.

      * **For a Repository**: `repo` (Full control of private repositories).
      * **For an Organization**: `admin:org`.
      * **For an Enterprise**: `admin:enterprise`.

3.  **Store the Token**: Copy the generated token and keep it somewhere secure. You will use it in the next step.

    > **Security Warning**: Treat this PAT like a password. Do not hardcode it in scripts or check it into version control. The next step will demonstrate how to store it securely as a Kubernetes secret.

-----

### Step 4: Deploy the Runner Scale Set

The runner scale set defines the configuration for the actual runner pods that will execute your workflow jobs.

1.  **Create a Kubernetes Secret for the PAT**: This is the recommended practice for handling sensitive credentials.

      * First, create the namespace where your runners will live.
        ```bash
        kubectl create namespace arc-runners
        ```
      * Next, create the secret containing your PAT.
        ```bash
        kubectl create secret generic github-pat \
          --from-literal=github_token=<PASTE-YOUR-PAT-HERE> \
          --namespace arc-runners
        ```
      * Verify that the secret was created successfully.
        ```bash
        kubectl get secrets -n arc-runners
        ```

2.  **Install the Runner Scale Set Chart**: This Helm chart deploys the `AutoScalingRunnerSet` resource that the controller will use to create runner pods.

    ```bash
    INSTALLATION_NAME="arc-runner-set"
    NAMESPACE="arc-runners"
    GITHUB_CONFIG_URL="https://github.com/your-org/your-repo" # Or https://github.com/your-org

    helm install "${INSTALLATION_NAME}" \
      --namespace "${NAMESPACE}" \
      --set githubConfigUrl="${GITHUB_CONFIG_URL}" \
      --set githubConfigSecret=github-pat \
      oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
    ```

    *Note: We are referencing the secret `github-pat` created earlier instead of passing the token directly.*

3.  **Updating Configuration with a Custom `values.yaml`**: To customize runner pod templates, scaling parameters, and more, use a `values.yaml` file and the `helm upgrade` command.

      * **Example `poc-values.yaml`**:
        ```yaml
        # poc-values.yaml
        minRunners: 1
        maxRunners: 5
        template:
          spec:
            containers:
              - name: runner
                image: ghcr.io/actions/actions-runner:latest
                resources:
                  requests:
                    cpu: 250m
                    memory: 512Mi
                  limits:
                    cpu: 500m
                    memory: 1Gi
        ```
      * **Dry Run**: To preview the changes without applying them, use the `--dry-run` flag.
        ```bash
        helm upgrade arc-runner-set \
          oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
          --namespace arc-runners \
          -f poc-values.yaml \
          --dry-run
        ```
      * **Apply Changes**: To apply the new configuration, run the command without `--dry-run`.
        ```bash
        helm upgrade arc-runner-set \
          oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
          --namespace arc-runners \
          -f poc-values.yaml
        ```

-----

### Troubleshooting: Debugging Failing Pods

If your runner pods are created but die too quickly to inspect logs, it often points to a network, configuration, or permissions issue. You can launch a temporary debug pod in the same namespace to investigate.

This command starts an interactive shell in a pod with network tools like `curl`.

```bash
kubectl run net-test -n arc-runners --rm -it \
  --image=alpine/curl -- /bin/sh
```

From inside this pod's shell, you can test connectivity to GitHub (`curl https://api.github.com`) or other internal services to diagnose network policies or DNS issues.
