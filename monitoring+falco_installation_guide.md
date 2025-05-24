# Installing Monitoring and Security Tools in KubeSphere

This guide details the installation of Grafana for visualization, configuration of Prometheus data sources for your application metrics (MySQL, Frontend, Backend), and the setup of Falco for runtime security monitoring within your KubeSphere cluster.

## Prerequisites

* A working KubeSphere cluster deployed on your VMs (as per the previous guide).
* SSH access to your Master node.
* `kubectl` configured to interact with your cluster (usually done automatically by KubeKey on the master).
* `helm` installed on the cluster.

## 1. Install Prometheus

Prometheus is the core monitoring engine used to collect metrics from the Kubernetes cluster.

* Log in to the KubeSphere web console with your credentials;
* Navigate to **KubeSphere Platform**.
* Search for **WhizardTelemetry Monitoring** and click Install.
    * Make sure to select the **recommended version**.
    * Select the **host nodes** where the service will be deployed.
* Wait for the **Cluster Agent** installation to complete (this may take a few minutes).

Now **Prometheus monitoring stack** is fully deployed and running on the cluster.
The final step is to configure external access in order to access the Prometheus dashboard.

* Return to the main dashboard of Kubesphere.
* Go to **Cluster Management** -> **host**.
* Choose **Services** form **Application Workloads**.
* Search for and open the `prometheus-k8s` service.
* Click on **More** -> **Edit External Access**.
* In the configuration window:
   * Set **Access Mode** to `NodePort`.
   * Save the changes.

By default, **Kubernetes** assigns a random range **NodePort** to services configured with NodePort access. To standardize access we have to set the **Prometheus NodePort** to `30090`. 

In the same `prometheus-k8s` service:
* Click on **More** -> **Edit YAML**.
* Scroll to line 21 (or locate the ports: section) and Replace the automatically assigned NodePort with `30090`.
* Click **Save** to apply the changes.

After saving, **Prometheus** will be accessible at:

```bash
http://<MasterNode_IP>:30090
```

This finalizes Prometheus installation and external access setup.




## 2. Install MySQL Exporter

Install the Prometheus MySQL Exporter using Helm to collect metrics from your MySQL database.

1. Add the Prometheus Community Helm repository:
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts/
   ```

2. Update Helm repositories:
   ```bash
   helm repo update
   ```

3. Install the MySQL Exporter:
   ```bash
   helm install mysql-exporter prometheus-community/prometheus-mysql-exporter --namespace bank-project --values mysql-export-values.yaml
   ```

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

### Configure Frontend Metrics

Your frontend service need to expose metrics in a format Prometheus understands (usually `/metrics` endpoint in the Prometheus text format). If they already do this, you need to tell Prometheus how to find them, often using a ServiceMonitor resource if you are using the Prometheus Operator (which is common in KubeSphere).

1. Apply ServiceMonitors: Apply the YAML files that define how Prometheus should scrape metrics from your frontend and backend services.
   ```bash
   kubectl apply -f service-monitor-frontend.yaml -n bank-project
   ```

2. Allow a couple of minutes for Prometheus to apply the new servicemonitor.


### Configure Backend Metrics

As done for the frontend, you need to apply a ServiceMonitor for the backend service as well.

1. Apply ServiceMonitors: Apply the YAML files that define how Prometheus should scrape metrics from your frontend and backend services.
   ```bash
   kubectl apply -f service-monitor-frontend.yaml -n bank-project
   ```

2. Allow a couple of minutes for Prometheus to apply the new servicemonitor.




### Import Dashboards into Grafana

Import pre-built Grafana dashboards designed to visualize the metrics collected from MySQL, Frontend, and Backend services.

1. Go to the Grafana dashboard in your browser.
2. Click "Dashboards" (or the four squares icon) in the left-hand side panel, then select "New" and "Import".
3. You will typically be prompted to upload a .json file or paste JSON text for the dashboard.
4. Import the JSON files for your mysql_dashboard.json, frontend_dashboard.json, and backend_dashboard.json one by one. During the import process, ensure you select the Prometheus data source you configured earlier when prompted.

After importing the dashboards and allowing Prometheus time to scrape the metrics, you should now be able to view the dashboards in Grafana populated with data from your applications.









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

4. Login using the default credentials: Username `admin`, Password `admin` (or the password you set with `--set adminPassword`).

### Configure Prometheus Data Source in Grafana

Before configuring Grafana, you need to make Prometheus accessible from outside the cluster:

1. Deploy the NodePortProm.yaml file to expose Prometheus via NodePort:
   ```bash
   kubectl apply -f NodePortProm.yaml
   ```

2. Once deployed, Prometheus will be accessible externally at `http://<MASTER_NODE_IP>:30090`

3. Now log into the Grafana dashboard, click "Connections" (or the plug icon) in the left-hand side panel, then select "Data Sources".

4. Click "Add data source" and choose "Prometheus" from the list.

5. In the "Settings" panel, find the "Connection" section.

6. For the "URL", enter `http://<MASTER_NODE_IP>:30090` (replacing `<MASTER_NODE_IP>` with your master node's IP address).

7. Scroll down and click the "Save & test" button.

8. You should see a green confirmation box stating "Successfully queried the Prometheus API."
## 3. Install and Configure Application Metrics

Now that Grafana is connected to Prometheus, you can expose your application's metrics so Prometheus can scrape them, and then visualize them in Grafana.
