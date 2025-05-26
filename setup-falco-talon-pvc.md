# Essential Guide: Persistent Log Storage for Falco Talon in Kubernetes

This guide shows you how to set up Falco Talon to save action logs to a **PersistentVolume (PV)** via a **PersistentVolumeClaim (PVC)**. This ensures your logs are persistent, meaning they won't be lost if the Falco Talon pod restarts or moves, and they'll be directly accessible from the cluster node.

---

## Prerequisites

* A running Kubernetes cluster.
* `kubectl` configured to access your cluster.
* Falco and Falco Talon already installed.
* An operational storage provisioner (specifically **`openebs.io/local`** in your case).

---

## Phase 1: Create the PersistentVolumeClaim (PVC)

First, you'll define the persistent storage request for Falco Talon. This PVC acts as a claim for storage space from your cluster.

Create a file named `falco-talon-logs-pvc.yaml` with the following content:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: falco-talon-logs-pvc # Name of your PVC.
  namespace: falco          # Same namespace as your Falco Talon Deployment.
spec:
  accessModes:
    - ReadWriteOnce         # Allows a single node to mount read/write.
  resources:
    requests:
      storage: 1Gi          # Requests 1 Gigabyte of storage. Adjust as needed.
  storageClassName: local   # **IMPORTANT:** Must match an available StorageClass in your cluster (e.g., "local" for OpenEBS).
```

**Apply the PVC:**
`kubectl apply -f falco-talon-logs-pvc.yaml`
### Phase 2: Modify Falco Talon's Deployment

Now, you'll update your Falco Talon Deployment to use the newly created PVC. You'll add references to the PVC in two places within your `Deployment.yaml` file.

**Open your `Deployment.yaml` file for Falco Talon and add the following sections:**

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: falco-talon
  namespace: falco
  labels:
    app.kubernetes.io/instance: falco-talon
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: falco-talon
    app.kubernetes.io/part-of: falco-talon
    app.kubernetes.io/version: 0.3.0
    helm.sh/chart: falco-talon-0.3.0
  annotations:
    deployment.kubernetes.io/revision: '5'
    meta.helm.sh/release-name: falco-talon
    meta.helm.sh/release-namespace: falco
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/instance: falco-talon
      app.kubernetes.io/name: falco-talon
  template:
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/instance: falco-talon
        app.kubernetes.io/name: falco-talon
      annotations:
        secret-checksum: 74234e98afe7498fb5daf1f36ac2d78acc339464f950703b8c019892f982b90b
    spec:
      volumes: # <--- This is the 'volumes' section of the pod. Add the PVC volume here.
        - name: rules
          configMap:
            name: falco-talon-rules
            defaultMode: 420
        - name: config
          secret:
            secretName: falco-talon-config
            defaultMode: 420
        # --- START ADDITION FOR PVC VOLUME ---
        - name: falco-talon-logs-volume # Internal name for this volume.
          persistentVolumeClaim:
            claimName: falco-talon-logs-pvc # **IMPORTANT: This must match the PVC name from Phase 1.**
        # --- END ADDITION FOR PVC VOLUME ---
      containers:
        - name: falco-talon
          image: 'falco.docker.scarf.sh/falcosecurity/falco-talon:0.3.0'
          args:
            - server
            - '-c'
            - /etc/falco-talon/config.yaml
            - '-r'
            - /etc/falco-talon/rules.yaml
          ports:
            - name: http
              containerPort: 2803
              protocol: TCP
            - name: nats
              containerPort: 4222
              protocol: TCP
          env:
            - name: LOG_LEVEL
              value: info
          resources: {}
          volumeMounts: # <--- This is the 'volumeMounts' section for the container. Mount the PVC volume here.
            - name: config
              readOnly: true
              mountPath: /etc/falco-talon/config.yaml
              subPath: config.yaml
            - name: rules
              readOnly: true
              mountPath: /etc/falco-talon/rules.yaml
              subPath: rules.yaml
            # --- START ADDITION FOR PVC VOLUME MOUNT ---
            - name: falco-talon-logs-volume # Must match the volume 'name' defined in the `volumes` section above.
              mountPath: /var/falco-talon-logs # **IMPORTANT: This is the path inside the container where Falco Talon will write logs.**
            # --- END ADDITION FOR PVC VOLUME MOUNT ---
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
              scheme: HTTP
            initialDelaySeconds: 10
            timeoutSeconds: 1
            periodSeconds: 5
            successThreshold: 1
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /healthz
              port: http
              scheme: HTTP
            initialDelaySeconds: 10
            timeoutSeconds: 1
            periodSeconds: 5
            successThreshold: 1
            failureThreshold: 3
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          imagePullPolicy: Always
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      dnsPolicy: ClusterFirst
      serviceAccountName: falco-talon
      serviceAccount: falco-talon
      securityContext:
        runAsUser: 1234
        fsGroup: 1234
      schedulerName: default-scheduler
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600
```
**Apply the Modified Deployment:**

Save the changes to your Deployment.yaml (ensure you're using the correct file name), then apply it to your cluster. Kubernetes will roll out the update, restarting the Falco Talon pod with the new volume configuration.

`kubectl apply -f your-falco-talon-deployment.yaml # Replace with your Deployment file name`
### Phase 3: Access the Saved Logs

Now that Falco Talon is configured to save logs to the PVC, and the PVC is powered by OpenEBS Local PV, here's how you can access those logs directly.

#### Method: Direct Node Access (Fastest for OpenEBS Local PV)

This method is efficient because your Local PV data is physically on the filesystem of the node where the PV is located.

1.  **Identify the PersistentVolume (PV) and its associated node/path:**
    First, find which node your `falco-talon-logs-pvc` is bound to and the specific path of the PV on that node.

    ```bash
    # Get the PV name associated with your PVC
    kubectl get pvc falco-talon-logs-pvc -n falco -o jsonpath='{.spec.volumeName}'

    # Then, describe the PV to find its node and path. Replace <PV_NAME_FROM_ABOVE> with the actual PV name.
    kubectl get pv pvc-62c0a4e7-dc4b-41fb-9eb1-87fe8d37a28b -o yaml | grep "path:"
    ```
    
2.  **SSH into the identified node:**
    Connect to the Kubernetes node you identified in the previous step (e.g., `worker1`) using SSH. You'll need the appropriate SSH user and the node's IP address or hostname.

    ```bash
    ssh <ssh_user>@<node_ip_or_hostname>
    # Example: ssh ubuntu@192.168.1.10 (if 'worker1' has IP 192.168.1.10)
    ```

3.  **Navigate and view logs on the node:**
    Once connected via SSH, you'll need to navigate to the PV's directory on the node, and then into the `falco-talon-logs/` subdirectory (which is the `mountPath` you specified in the Falco Talon Deployment).

    ```bash
    sudo su - # You might need root privileges to access /var/openebs directories
    cd /<PV_PATH_ON_NODE_FROM_STEP_1>/falco-talon-logs/
    # Example: cd /var/openebs/local/pv-62c0a4e7-dc4b-41fb-9eb1-87fe8d37a28b/falco-talon-logs/

    ls -l                     # List all log files present
    cat <log_file_name>       # Read the content of a specific log file
    tail -f <log_file_name>   # Follow new log entries in real-time
    ```
    When you're finished, type `exit` twice to leave the root session and then your SSH connection.
