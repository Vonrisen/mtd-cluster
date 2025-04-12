The entire setup was carried out using the VMWare hypervisor with four virtual machines, all configured in **bridged** mode:

- **Ubuntu Server** (for cluster management)
    
- **Master Node** (for the control plane) with TalosOS
    
- **Worker1 Node** (Worker Node) with TalosOS
    
- **Worker2 Node** (Worker Node) with TalosOS
    

**All operations will be performed from the Ubuntu Server machine.** Let’s begin by downloading the `talosctl` tool, which allows us to create and interact with our future cluster: [https://www.talos.dev/v1.9/talos-guides/install/talosctl/](https://www.talos.dev/v1.9/talos-guides/install/talosctl/)

Once downloaded, create a folder to contain our cluster configuration files. We'll call it `my-cluster`:

```bash
mkdir my-cluster
```

Generate the configuration files using the following command:

```bash
talosctl gen config <CLUSTER_NAME> https://<MASTER_IP>:6443 --output-dir ./<DESTINATION_PATH>
```

In my case:

```bash
talosctl gen config my-cluster https://192.168.1.106:6443 --output-dir ./my-cluster
```

This will generate three files:

- `controlplane.yaml`
    
- `worker.yaml`
    
- `talosconfig`
    

The `.yaml` files are the cluster configuration files. The `talosconfig` file is a local configuration file used to connect and authenticate access to the cluster.

To improve readability, you can remove the comment lines from the configuration files using:

```bash
sed -i.bak '/^\s*#/d; /^\s*$/d' controlplane.yaml worker.yaml
```

This command will also create backup files containing the original content with comments.

## Editing the Configuration Files

### Editing `controlplane.yaml`

Under the `machine.network` section, edit the configuration as follows:

```yaml
network:
  hostname: master
  interfaces:
    - interface: <NETWORK_INTERFACE>
      dhcp: false
      addresses:
        - <MASTER_IP>/24
      routes:
        - network: 0.0.0.0/0
          gateway: <GATEWAY_IP>
  nameservers:
    - 1.1.1.1
    - 8.8.8.8
```

To identify your network interface, run:

```bash
ip -brief a
```

Example output:

```bash
ens33            UP             192.168.1.105/24 ...
```

In this case, update your configuration as follows:

```yaml
network:
  hostname: master
  interfaces:
    - interface: ens33
      dhcp: false
      addresses:
        - 192.168.1.106/24
      routes:
        - network: 0.0.0.0/0
          gateway: 192.168.1.254
  nameservers:
    - 1.1.1.1
    - 8.8.8.8
```

To discover your gateway IP:

```bash
ip route | awk '/default/ {print $3}'
```

### Creating Worker Configurations

Since we have two worker nodes with static IPs, we need two configuration files:

```bash
mv worker.yaml worker1.yaml
cp worker1.yaml worker2.yaml
```

Update both `worker1.yaml` and `worker2.yaml` with the appropriate settings. Here's the structure:

```yaml
network:
  hostname: <WORKER_NAME>
  interfaces:
    - interface: <NETWORK_INTERFACE>
      dhcp: false
      addresses:
        - <WORKER_IP>/24
      routes:
        - network: 0.0.0.0/0
          gateway: <GATEWAY_IP>
  nameservers:
    - 1.1.1.1
    - 8.8.8.8
```

Example for `worker1.yaml`:

```yaml
network:
  hostname: worker1
  interfaces:
    - interface: ens33
      dhcp: false
      addresses:
        - 192.168.1.107/24
      routes:
        - network: 0.0.0.0/0
          gateway: 192.168.1.254
  nameservers:
    - 1.1.1.1
    - 8.8.8.8
```

## Applying the Configuration

To apply the configuration to each node:

```bash
talosctl apply-config --insecure --nodes <NODE_IP> --file <CONFIGURATION_FILE_PATH>
```

Example for the master node:

```bash
talosctl apply-config --insecure --nodes 192.168.1.106 --file controlplane.yaml
```

### Troubleshooting

If you receive an error like:

```text
rpc error: code = Unknown desc = failed to parse config: decode error: yaml: line 16: found character that cannot start any token
```

It likely means there's an indentation error. Check that you're using tabs consistently and that the structure is correctly aligned.

## Updating `talosconfig`

Edit `talosconfig` and add the master and worker nodes IP under `contexts.<CLUSTER_NAME>`:

```yaml
contexts:
  my-cluster:
    endpoints:
      - 192.168.1.106
    nodes:
      - 192.168.1.106
      - 192.168.1.107
      - 192.168.1.108
```

## Bootstrapping the Cluster

Now bootstrap the cluster:

```bash
talosctl --talosconfig <TALOSCONFIG_PATH> bootstrap --nodes <MASTER_IP>
```

Example:

```bash
talosctl --talosconfig talosconfig bootstrap --nodes 192.168.1.106
```

If everything works fine, the TalosOS nodes are correctly set up.

## Installing kubectl

Follow the official Kubernetes documentation: [https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

Once installed, generate the kubeconfig file:

```bash
talosctl kubeconfig --talosconfig <TALOSCONFIG_PATH> --nodes <MASTER_IP>
```

Now check your nodes:

```bash
kubectl get nodes
```

Example output:

```text
NAME      STATUS   ROLES           AGE     VERSION
master    Ready    control-plane   9m33s   v1.32.3
worker1   Ready    <none>          9m30s   v1.32.3
worker2   Ready    <none>          9m32s   v1.32.3
```

### Adding Roles to Workers (Optional)

You can assign a role to the worker nodes for better readability:

```bash
kubectl label node <NODE_NAME> node-role.kubernetes.io/worker=
```

Example:

```bash
kubectl label node worker2 node-role.kubernetes.io/worker=
```

After applying the labels:

```bash
kubectl get nodes
```

```text
NAME      STATUS   ROLES           AGE   VERSION
master    Ready    control-plane   12m   v1.32.3
worker1   Ready    worker          12m   v1.32.3
worker2   Ready    worker          12m   v1.32.3
```

## Environment Variable Setup (Recommended)

To avoid having to specify the `--talosconfig` flag every time, define it as an environment variable:

```bash
echo 'export TALOSCONFIG="<TALOSCONFIG_ABSOLUTE_PATH>"' >> ~/.bashrc && source ~/.bashrc
```

Example:

```bash
echo 'export TALOSCONFIG="/home/ubuntu/my-cluster/talosconfig"' >> ~/.bashrc && source ~/.bashrc
```

---

You now have a working TalosOS Kubernetes cluster with proper configuration and accessibility!
