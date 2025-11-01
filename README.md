# Deploying the Grafana Loki Stack on Minikube

This guide provides a complete walkthrough for deploying a production-style, persistent logging stack on a local Minikube cluster. This setup uses:

  * **Loki:** The main log storage and query engine.
  * **Promtail:** The agent responsible for scraping logs and sending them to Loki.
  * **Grafana:** The UI for visualizing and querying logs.
  * **Helm:** The package manager for deploying these components to Kubernetes.

This configuration is ideal for local development, testing LogQL, and learning Loki's architecture.

## Prerequisites

Before you begin, you must have the following tools installed on your local machine:

  * [Docker](https://docs.docker.com/get-docker/) (or a compatible container runtime like Colima or Podman)
  * [Minikube](https://minikube.sigs.k8s.io/docs/start/)
  * [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
  * [Helm](https://helm.sh/docs/intro/install/)

-----

##  Step 1: Start Minikube

First, start your Minikube cluster. We'll give it 4GB of memory to ensure we have enough room for Grafana and the Loki components.

```bash
# Start a new Minikube cluster using the Docker driver
minikube start 
```

**Verify your cluster is running:**

```bash
kubectl get nodes
```

You should see your `minikube` node in a `Ready` state.

-----

##  Step 2: Configure Helm

Next, add the Grafana Helm repository, which contains the charts for Loki, Promtail, and Grafana.

```bash
# Add the Grafana chart repository
helm repo add grafana https://grafana.github.io/helm-charts

# Fetch the latest updates from all repositories
helm repo update
```

-----

##  Step 3: Deploy Loki (The Server)

We will deploy Loki in `SingleBinary` (monolithic) mode, but with persistent `filesystem` storage. This requires a specific `values.yaml` to handle chart validation and resource constraints on Minikube.

### 3.1. Refer Loki Configuration (`loki-values.yaml`)

This file is carefully configured to:

1.  Run in `SingleBinary` mode.
2.  Use `filesystem` storage (not S3).
3.  Provide "dummy" `bucketNames` to satisfy Helm chart validation quirks.
4.  Enable the `compactor` for log retention (set to 7 days).
5.  Lower the `chunksCache` memory to fit on Minikube.

<!-- end list -->


### 3.2. Install Loki

Create a dedicated `loki` namespace and install the chart.

```bash
kubectl create namespace loki
helm install loki grafana/loki -n loki -f loki-values.yaml
```

### 3.3. Verify Loki Deployment

It may take a minute for all pods to become `Running`.

```bash
kubectl get pods -n loki -w
```

You should see the following pods all in a `Running` state:

  * `loki-0` (The main StatefulSet)
  * `loki-canary-xxxxx` (Health checker)
  * `loki-chunks-cache-0` (Memcached for chunk caching)
  * `loki-gateway-xxxxx` (NGINX front-door)
  * `loki-results-cache-0` (Memcached for query result caching)

-----

## Step 4: Deploy Promtail (The Agent)

Now we deploy the Promtail agent to scrape logs and send them to Loki. We deploy it from its own chart.

### 4.1. Check Promtail Configuration (`promtail-values.yaml`)

This file tells Promtail where to find the Loki service we just deployed (`loki-gateway`). It also includes best-practice filters to drop noisy health-check logs.


### 4.2. Install Promtail

We install Promtail into the *same* `loki` namespace.

```bash
helm install promtail grafana/promtail -n loki -f promtail-values.yaml
```

### 4.3. Verify Promtail Deployment

Promtail runs as a `DaemonSet` (one pod on every node).

```bash
# 1. Check that the DaemonSet is created
kubectl get daemonset -n loki

# 2. Check that the new promtail-xxxxx pod is running
kubectl get pods -n loki -w
```

You will now see a `promtail-xxxxx` pod running alongside your Loki pods.

-----

##  Step 5: Deploy Grafana (The UI)

Finally, we install Grafana and automatically configure it to use our Loki deployment as its default data source.

### 5.1. Check Grafana Configuration (`grafana-values.yaml`)

This file defines the Loki data source, pointing it to the `loki-gateway` service.


### 5.2. Install Grafana

We'll put Grafana in its own `grafana` namespace.

```bash
kubectl create namespace grafana
helm install grafana grafana/grafana -n grafana -f grafana-values.yaml
```

### 5.3. Get Grafana Admin Password

The Helm chart generates a random admin password. Run this command to retrieve it:

```bash
echo "Grafana Admin Password:"
kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

**Copy the password that is printed\!**

-----

##  Step 6: Access and Use Your Logging Stack

Your entire stack is deployed. Now let's access it.

### 6.1. Access Grafana

1.  **Open a new, separate terminal** and run this `port-forward` command. It will run until you stop it (Ctrl+C).

    ```bash
    kubectl port-forward svc/grafana 3000:80 -n grafana
    ```

2.  Open your browser and go to: `http://localhost:3000`

3.  Log in:

      * **Username:** `admin`
      * **Password:** (The password you copied in Step 5.3)

4.  Click the **Explore** icon (compass) on the left menu. The "Loki (Minikube)" data source will be selected. You can now run LogQL queries like `{namespace="loki"}` to see your logs\!

-----

##  Maintenance & Helper Commands

Here are useful commands for managing your deployment.

### View All Logging Pods

```bash
# See pods in both namespaces
kubectl get pods -n loki,grafana
```

### View Pod Logs (e.g., to debug Loki)

```bash
# Tail the logs from the main loki-0 pod
kubectl logs -f loki-0 -n loki

# If loki-0 isn't starting, check its sidecar
kubectl logs -f loki-0 -n loki -c loki-config
```

### Upgrade a Deployment

If you modify a `values.yaml` file, you don't need to uninstall. Just run `helm upgrade`.

```bash
# Example: Apply a change to your loki-values.yaml
helm upgrade loki grafana/loki -n loki -f loki-values.yaml

# Example: Apply a change to your promtail-values.yaml
helm upgrade promtail grafana/promtail -n loki -f promtail-values.yaml
```

### Uninstall the Stack

To tear everything down:

```bash
# Uninstall the Helm releases
helm uninstall loki -n loki
helm uninstall promtail -n loki
helm uninstall grafana -n grafana

# Delete the namespaces (this also deletes the PVCs)
kubectl delete namespace loki
kubectl delete namespace grafana
```