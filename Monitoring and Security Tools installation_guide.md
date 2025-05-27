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
* Navigate to **KubeSphere Marketplace**.
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
   * **Save** the changes.

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




## 2. Application Metrics

### Configure MySQL Metrics

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
   kubectl apply -f frontend-servicemonitor.yaml -n bank-project
   ```

2. Allow a couple of minutes for Prometheus to apply the new servicemonitor.


### Configure Backend Metrics

As done for the frontend, you need to apply a ServiceMonitor for the backend service as well.

1. Apply ServiceMonitors: Apply the YAML files that define how Prometheus should scrape metrics from your frontend and backend services.
   ```bash
   kubectl apply -f backend-servicemonitor.yaml -n bank-project
   ```

2. Allow a couple of minutes for Prometheus to apply the new servicemonitor.








## 3. Install Grafana

Grafana is a popular open-source platform for monitoring and observability, allowing you to create dashboards to visualize your metrics.


* Log in to the KubeSphere web console with your credentials;
* Navigate to **KubeSphere Marketplace**.
* Search for **Grafana for WhizardTelemetry** and click Install.
    * Make sure to select the **recommended version**.
    * Select the **host nodes** where the service will be deployed.
* Wait for the **Cluster Agent** installation to complete (this may take a few minutes).


Now **Grafana** is fully deployed and running on the cluster.
The final step is to configure external access in order to access the Grafana dashboard.

* Return to the main dashboard of Kubesphere.
* Go to **Cluster Management** -> **host**.
* Choose **Services** form **Application Workloads**.
* Search for and open the `grafana` service.
* Click on **More** -> **Edit External Access**.
* In the configuration window:
   * Set **Access Mode** to `NodePort`.
   * **Save** the changes.


After saving, **Grafana** will be accessible at:

```bash
http://<MasterNode_IP>:<NODE_PORT>
```

This finalizes Gragana installation and external access setup.


### Access Grafana Dashboard

1. By default you do not need to log in to see the dashboards contained in kube-prometheus-stack. 

1. Login using the default credentials: Username `admin`, Password `admin`

### Import Dashboards into Grafana

Import pre-built Grafana dashboards designed to visualize the metrics collected from MySQL, Frontend, and Backend services.

1. Go to the Grafana dashboard in your browser.
2. Click "Dashboards" (or the four squares icon) in the left-hand side panel, then select "New" and "Import".
3. You will typically be prompted to upload a .json file or paste JSON text for the dashboard.
4. Import the JSON files for your mysql_dashboard.json, frontend_dashboard.json, and backend_dashboard.json one by one. During the import process, ensure you select the Prometheus as data source.

After importing the dashboards and allowing Prometheus time to scrape the metrics, you should now be able to view the dashboards in Grafana populated with data from your applications.


## 2. Falco + Falco Sidekick + Falco Talon

Falco is a runtime security engine for Kubernetes. Sidekick enables sending security events to various targets.


### Installation Falco + Falco Sidekick


* Add the falco Helm chart repository from the :

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
```

* Install falcon and falcon sidekick via the command:

```bash
helm install falco falcosecurity/falco --namespace falco \
  --create-namespace \
  --set tty=true \
  --set falcosidekick.enabled=true \
  --set falcosidekick.webui.enabled=true \
  --set falcosidekick.webui.service.type=NodePort \
  --set falcosidekick.config.talon.address=http://falco-talon:2803
```

With these commands, we install **falco** and **falco sidekick** with the corresponding **UI** and configure the sending of information to **falco-talon**.

It's now time to configure external access in order to access the **Falco Sidekick UI**.

* Return to the main dashboard of Kubesphere.
* Go to **Cluster Management** -> **host**.
* Choose **Services** form **Application Workloads**.
* Search for and open the `falco-falcosidekick-ui` service.
* Click on **More** -> **Edit External Access**.
* In the configuration window:
   * Set **Access Mode** to `NodePort`.
   * **Save** the changes.

After saving, **Falco Sidekick Interface** will be accessible at:

```bash
http://<MasterNode_IP>:<NODE_PORT>
``` 
The credentials to log in are:
   * user: admin
   * password: admin

### Installation Falco Talon

Since we already have the remote repository configured we just need to update it:

```bash
helm repo update falcosecurity
``` 

Now, just deploy **falcosecurity/falco-talon** chart:

```bash
helm upgrade --install falco-talon falcosecurity/falco-talon --namespace falco
``` 

After deploying, you can check if pods are running properly:

```bash
kubectl get pods -n falco | grep falco-talon
```




