# Installing Monitoring and Security Tools in KubeSphere

This guide details the installation of Grafana for visualization, configuration of Prometheus data sources for your application metrics (MySQL, Frontend, Backend), and the setup of Falco for runtime security monitoring within your KubeSphere cluster.

## Prerequisites

* A working KubeSphere cluster deployed on your VMs (as per the previous guide).
* SSH access to your Master node.
* `kubectl` configured to interact with your cluster (usually done automatically by KubeKey on the master).
* Helm installed on your Master node.

## 1. Install Helm

Helm is a package manager for Kubernetes, used to deploy applications packaged as Charts. We will use Helm to install Grafana, Prometheus Exporters, and Falco.

Follow the official Helm documentation to install the Helm binary on your Master node:

* [Official Helm Installation Guide](https://helm.sh/docs/intro/install/)

Once installed, verify Helm is working by checking its version:

```bash
helm version
```

## 2. Install and Configure Grafana

Grafana is a popular open-source platform for monitoring and observability, allowing you to create dashboards to visualize your metrics.

### Add Grafana Helm Repository

Add the official Grafana Helm chart repository:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
```

### Update Helm Repositories

Fetch the latest charts from all added repositories:

```bash
helm repo update
```

### Install Grafana using Helm

Install Grafana into a dedicated namespace called monitoring. We will expose the Grafana UI using a NodePort service for easy access in a demo environment.

```bash
helm install grafana grafana/grafana \
  --namespace monitoring --create-namespace \
  --set adminPassword='admin' \
  --set service.type=NodePort
```

* `--namespace monitoring --create-namespace`: Creates the monitoring namespace if it doesn't exist and installs Grafana into it.
* `--set adminPassword='admin'`: Sets the initial password for the admin user to admin. Warning: This is highly insecure for any environment beyond a purely isolated demo. Change this password immediately after logging in.
* `--set service.type=NodePort`: Exposes the Grafana UI service on a static port across all your cluster nodes, making it accessible from outside the cluster.

### Access Grafana Dashboard

1. Wait a few moments for Grafana pods to start. Check their status:
   ```bash
   kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana
   ```
   Wait until the pod shows a Running status.

2. Find the NodePort assigned to the Grafana service:
   ```bash
   kubectl get services -n monitoring grafana
   ```
   Look for the NODEPORT listed in the output (e.g., 30880:30000/TCP).

3. Open your web browser and navigate to `http://<Master_Static_IP>:<Grafana_NodePort>`. Replace `<Master_Static_IP>` with the actual static IP address of your Master node and `<Grafana_NodePort>` with the NodePort you found in the previous step.

4. Login using the default credentials: Username `admin`, Password `admin` (or the password you set with `--set adminPassword`). Change this password immediately after logging in!

### Configure Prometheus Data Source in Grafana

KubeSphere typically includes a Prometheus instance for cluster monitoring. You need to configure Grafana to use this Prometheus as a data source to visualize metrics.

1. Once logged into the Grafana dashboard, click "Connections" (or the plug icon) in the left-hand side panel, then select "Data Sources".
2. Click "Add data source" and choose "Prometheus" from the list.
3. In the "Settings" panel, find the "Connection" section.
4. For the "URL", you need the in-cluster service address of your Prometheus instance. If you are using the default Prometheus installed by KubeSphere, it's typically named prometheus-k8s and resides in the kubesphere-monitoring-system or monitoring namespace, exposed on port 9090. The typical in-cluster URL format is `http://<service-name>.<namespace>.svc:<port>`.
5. You can verify the service name and namespace using:
   ```bash
   kubectl get services -A | grep prometheus
   ```
   A common URL for KubeSphere's Prometheus would be:
   `http://prometheus-k8s.kubesphere-monitoring-system.svc:9090`
6. Enter the correct Prometheus service URL in the URL field.
7. Scroll down and click the "Save & test" button.
8. You should see a green confirmation box stating "Successfully queried the Prometheus API."

## 3. Install and Configure Application Metrics

Now that Grafana is connected to Prometheus, you can expose your application's metrics so Prometheus can scrape them, and then visualize them in Grafana.

### Install MySQL Exporter

Install the Prometheus MySQL Exporter using Helm to collect metrics from your MySQL database.

1. Add the Prometheus Community Helm repository:
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts/
   ```

2. Update Helm repositories:
   ```bash
   helm repo update
   ```

3. Install the MySQL Exporter. You will need a values file (`<path_to_your_values_file>`) containing the connection details for your MySQL database.
   ```bash
   helm install mysql-exporter prometheus-community/prometheus-mysql-exporter --namespace bank-project --values <path_to_your_values_file>
   ```
   * `--namespace bank-project`: Installs the exporter into the bank-project namespace (assuming your banking application is in this namespace). Ensure this namespace exists (`kubectl create namespace bank-project`).
   * `--values <path_to_your_values_file>`: This is crucial. Your values file must contain configuration allowing the exporter to connect to your MySQL database. This typically includes the MySQL host, port, username, and password (often stored securely using Kubernetes Secrets referenced in the values file). Consult the prometheus-mysql-exporter chart documentation for details on configuring database access in the values file.

4. Upon successful installation, you will see output similar to the one you provided:
   ```
   NAME: mysql-exporter
   LAST DEPLOYED: Mon Apr 28 10:04:10 2025 # Date/time will vary
   NAMESPACE: bank-project
   STATUS: deployed
   REVISION: 1
   TEST SUITE: None
   NOTES:
   1. Get the application URL by running these commands:
      export POD_NAME=$(kubectl get pods --namespace bank-project -l "app.kubernetes.io/name=prometheus-mysql-exporter,app.kubernetes.io/instance=mysql-exporter" -o jsonpath="{.items[0].metadata.name}")
      echo "Visit http://127.0.0.1:9104 to use your application"
      kubectl --namespace bank-project port-forward $POD_NAME 9104
   ```
   The NOTES section often provides access methods, but for integrating with Prometheus within the cluster, the exporter's service is what Prometheus will scrape, not typically accessed via port-forward.

### Configure Frontend and Backend Metrics

Your frontend and backend services need to expose metrics in a format Prometheus understands (usually `/metrics` endpoint in the Prometheus text format). If they already do this, you need to tell Prometheus how to find them, often using a ServiceMonitor resource if you are using the Prometheus Operator (which is common in KubeSphere).

1. Apply ServiceMonitors: Apply the YAML files that define how Prometheus should scrape metrics from your frontend and backend services.
   ```bash
   kubectl apply -f service-monitor-frontend.yaml -n bank-project
   kubectl apply -f service-monitor-backend.yaml -n bank-project # Assuming you have a similar file for the backend
   ```
   Note: You need to have `service-monitor-frontend.yaml` and `service-monitor-backend.yaml` files. These files define ServiceMonitor objects that select the Kubernetes Service(s) for your frontend and backend applications and specify the path and port where metrics are exposed. Ensure these files correctly target your services in the bank-project namespace.

2. Allow a couple of minutes for Prometheus to discover these new ServiceMonitor targets and start scraping metrics.

### Import Dashboards into Grafana

Import pre-built Grafana dashboards designed to visualize the metrics collected from MySQL, Frontend, and Backend services.

1. Go to the Grafana dashboard in your browser.
2. Click "Dashboards" (or the four squares icon) in the left-hand side panel, then select "New" and "Import".
3. You will typically be prompted to upload a .json file or paste JSON text for the dashboard.
4. Import the JSON files for your mysql_dashboard.json, frontend_dashboard.json, and backend_dashboard.json one by one. During the import process, ensure you select the Prometheus data source you configured earlier when prompted.

After importing the dashboards and allowing Prometheus time to scrape the metrics, you should now be able to view the dashboards in Grafana populated with data from your applications.

## 4. Install Falco for Runtime Security Monitoring

Falco is a behavioral activity monitor designed to detect anomalous activity in your applications and infrastructure in real-time. We will install it using its Helm chart.

### Add Falco Security Helm Repository

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
```

### Update Helm Repositories

```bash
helm repo update
```

### Install Falco

Install Falco into a dedicated namespace. The `--replace` flag can be used if you need to reinstall, but be cautious as it will delete and recreate resources.

```bash
helm install --replace falco --namespace falco --create-namespace --set tty=true falcosecurity/falco
```
* `--namespace falco --create-namespace`: Creates the falco namespace if it doesn't exist and installs Falco into it.
* `--set tty=true`: This setting might be necessary in some containerized environments or specific kernel configurations for Falco to function correctly.

### Wait for Falco Pods

Wait for the Falco pods to start and become Running. You can monitor their status:

```bash
watch kubectl get pods -n falco
```
Press Ctrl + C to exit the watch command once all Falco pods are running.

### Install Falco Sidekick

Falco Sidekick is a companion service that routes Falco alerts to various outputs. We will configure it to expose metrics that Prometheus can scrape.

```bash
helm install falcosidekick falcosecurity/falcosidekick \
  --namespace falco \
  --set webui.enabled=false \
  --set config.prometheus.enabled=true
```
* `--set webui.enabled=false`: Disables the Falco Sidekick web UI.
* `--set config.prometheus.enabled=true`: Configures Falco Sidekick to expose metrics in a Prometheus-compatible format.

### Configure Prometheus to Scrape Falco Sidekick Metrics

Apply a ServiceMonitor resource to tell Prometheus how to scrape the metrics exposed by Falco Sidekick.

```bash
kubectl apply -f falcosidekick-servicemonitor.yaml -n falco
```
Note: You need the `falcosidekick-servicemonitor.yaml` file. This file defines a ServiceMonitor that selects the Falco Sidekick service in the falco namespace and specifies the path/port for metrics (typically `/metrics` on port 2626).

After these steps, Prometheus will start scraping metrics from Falco Sidekick, and you can potentially import a Falco dashboard into Grafana (if available) or build your own to visualize security events or Falco's operational metrics.

You should now have a robust monitoring setup with Grafana visualizing application metrics via Prometheus and Falco running for runtime security analysis.
