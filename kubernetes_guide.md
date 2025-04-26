# Guide to Installing KubeSphere for a Banking Application Demo

This guide outlines the steps to install KubeSphere on a set of virtual machines, suitable for deploying and managing a banking application demo environment. We will use KubeKey (`kk`) to deploy Kubernetes and KubeSphere.

**Target Environment:**

* Minimum 3 Virtual Machines (VMs): 1 Master node, 2 Worker nodes.
* All VMs configured with static IP addresses.
* All VMs connected to the same network, using a "Bridged" network configuration.

## 1. Prerequisites: VM Setup and Initial Configuration

Before installing KubeSphere, you need to set up your virtual machines and install necessary prerequisites on *each* node.

1.  **VM Creation and Static IP Setup**:
    * Create your three virtual machines (Master, Worker1, Worker2) using your preferred hypervisor (VMware, VirtualBox, etc.).
    * **Resource Allocation**: For a demo environment, the following resources are recommended for *each* VM. Adjust these based on the expected workload of your banking application:
        * **RAM**: Minimum 4GB
        * **Storage**: Minimum 40GB
        * **CPUs**: Minimum 2 Processors / 2 Cores per Processor (Total 4 vCPUs)
    * **Network Configuration**: Ensure all three VMs are configured with a **"Bridged" network adapter**. This mode allows the VMs to communicate directly with your physical network, receiving an IP address from your router (or physical network's DHCP server) and making them accessible from your local machine and each other using their static IPs.
    * **Static IP Assignment**: Crucially, configure a **static IP address** for each VM using Netplan or your distribution's equivalent network configuration tool. Refer to our previous guide for detailed steps: [Link to your previous Static IP guide here]
    * Ensure the Master node can connect to the Worker nodes via SSH using the username and password (or SSH keys) you will specify in the KubeKey configuration later.

2.  **Install Essential Packages**: On *each* of the three VMs (Master, Worker1, Worker2), install the following packages:
    * `conntrack`: Used by Kubernetes networking components (like kube-proxy) for connection tracking.
    * `socat`: A network utility often used for port forwarding and other communication tasks by Kubernetes.
    Execute the following command on **each** VM:
    ```bash
    sudo apt update
    sudo apt install -y conntrack socat
    ```

3.  **Install Docker Engine**: KubeSphere requires a container runtime, and Docker is a common choice. Install Docker Engine on *each* of the three VMs by following the official Docker installation guide. **It is highly recommended to follow the official documentation for the most up-to-date instructions:**
    * [Official Docker Engine Installation Guide for Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
    After installing Docker on each node, verify the service is running:
    ```bash
    sudo systemctl status docker
    ```
    It should show `Active: active (running)`.

At this point, all your VMs should be set up with static IPs, essential packages, and the Docker runtime installed. You are ready to proceed with the KubeSphere installation.

## 2. Installing KubeSphere with KubeKey

KubeKey (`kk`) is a powerful command-line tool used to install Kubernetes clusters and KubeSphere. You will perform the following steps from the **Master node**.

1.  **Download KubeKey**: Connect to your Master node via SSH. Download the KubeKey binary:
    ```bash
    curl -sfL [https://get-kk.kubesphere.io](https://get-kk.kubesphere.io) | VERSION=v3.0.13 sh -
    ```
    This command downloads a script that fetches the `kk` binary. The `VERSION=v3.0.13` part specifies the version of the `kk` tool itself (KubeKey), not the KubeSphere or Kubernetes version it will install.

2.  **Make KubeKey Executable**: Give execution permissions to the downloaded `kk` binary:
    ```bash
    chmod +x kk
    ```

3.  **Generate Cluster Configuration File**: KubeKey uses a YAML file to define the cluster structure and installation parameters. Generate a sample configuration file:
    ```bash
    ./kk create config --with-kubernetes v1.23.10 --with-kubesphere v3.4.1
    ```
    * `--with-kubernetes v1.23.10`: Specifies the version of Kubernetes to install.
    * `--with-kubesphere v3.4.1`: Specifies the version of KubeSphere to install on top of Kubernetes.
    This command will create a file named `config-sample.yaml` (or similar) in your current directory.

4.  **Edit the Cluster Configuration File**: Open the generated configuration file using a text editor (like `nano`):
    ```bash
    sudo nano config-sample.yaml
    ```
    You need to modify the `hosts` and `roleGroups` sections to define your nodes and their roles. Find the `spec:` section and locate the `hosts` and `roleGroups` blocks.

    Modify the `hosts` list to include details for your Master, Worker1, and Worker2 nodes. Then, configure the `roleGroups` to assign the correct roles.

    **Original Structure (Example):**

    ```yaml
    spec:
      hosts:
      - {name: <master_node_name>, address: <master_ip>, internalAddress: <master_ip>, user: <master_username>, password: "<password>"}
      - {name: <worker_name>, address: <worker_ip>, internalAddress: <worker_ip>, user: <worker_username>, password: "<password>"}
      # Add more worker entries as needed

      roleGroups:
        etcd:
        - <master_node_name>
        control-plane:
        - <master_node_name>
        worker:
        - <worker_name_1>
        # Add more worker names as needed
    # ... rest of the file
    ```

    **Example Modification (using your provided example):**

    ```yaml
    spec:
      hosts:
      - {name: master, address: 192.168.1.111, internalAddress: 192.168.1.111, user: master, password: "your_master_password"} # Replace with your actual details
      - {name: worker1, address: 192.168.1.112, internalAddress: 192.168.1.112, user: worker1, password: "your_worker1_password"} # Replace with your actual details
      - {name: worker2, address: 192.168.1.113, internalAddress: 192.168.1.113, user: worker2, password: "your_worker2_password"} # Replace with your actual details

      roleGroups:
        etcd:
        - master # Your master node will typically run etcd
        control-plane:
        - master # Your master node will be the control plane
        worker:
        - worker1 # List your worker node names here
        - worker2 # List your worker node names here
    # ... rest of the file
    ```
    * **Replace Placeholders**: Change `<master_node_name>`, `<master_ip>`, `<master_username>`, `<worker_name>`, `<worker_ip>`, `<worker_username>`, and `"<password>"` with your actual VM hostnames, static IP addresses, SSH usernames, and SSH passwords.
    * **Security Warning**: Using passwords directly in this configuration file is convenient for demos but **highly insecure** for any environment exposed to external networks. For better security, consider using SSH keys instead of passwords and configuring KubeKey to use them (refer to the KubeKey documentation for SSH key configuration). If using passwords, ensure they are strong and change them immediately after installation if possible.
    * **Roles**:
        * `etcd`: The distributed key-value store for the cluster state. Typically runs on control-plane nodes.
        * `control-plane`: Runs the Kubernetes control plane components (API server, scheduler, controller manager). Your Master node will have this role.
        * `worker`: Nodes where your application containers (pods) will run. Your Worker VMs will have this role.

    Save the changes to the file (`Ctrl + X`, then `Y`, then `Enter` in `nano`).

5.  **Create the Cluster**: Now, initiate the installation process using KubeKey and the modified configuration file:
    ```bash
    ./kk create cluster -f config-sample.yaml
    ```
    This command will start the deployment. KubeKey will connect to each node via SSH, install Kubernetes components, and then install KubeSphere. This process can take some time depending on your network speed and VM resources.

6.  **Monitor and Access KubeSphere**:
    * Watch the terminal output for progress messages and any errors.
    * Once the command completes successfully, KubeSphere should be installed.
    * KubeSphere is typically accessible via a NodePort service on port `30880` (by default) on any of the nodes. Connect your web browser to the Master node's static IP address on this port:
        ```
        https://<Master_Static_IP>:30880
        ```
        (Note: It's `https`). You might get a certificate warning, which is normal for a self-signed certificate used by default.
    * The default login credentials for the KubeSphere web console are usually username `admin` and password `P@88w0rd`.
    * **Strongly Recommended**: Log in immediately and change the default `admin` password for security reasons.

## Conclusion

You have successfully installed a Kubernetes cluster with KubeSphere on your VMs using KubeKey. You can now explore the KubeSphere console to deploy and manage your banking application demo, monitor resources, manage users, and more. Remember that the allocated resources are suitable for a demo; production environments would require significantly more planning and resources.
