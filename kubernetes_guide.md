# Guide to Installing KubeSphere for a Banking Application Demo

This guide outlines the steps to install KubeSphere on a set of virtual machines, suitable for deploying and managing a banking application demo environment. We will use KubeKey (`kk`) to deploy Kubernetes and KubeSphere.

**Target Environment:**

* Minimum 3 Virtual Machines (VMs): 1 Master node, 2 Worker nodes.
* All VMs configured with static IP addresses.
* All VMs connected to the same network, preferably using a "Bridged" network configuration.

---

## 1. Prerequisites: VM Setup and Initial Configuration

Before installing KubeSphere, you need to set up your virtual machines and install necessary prerequisites on *each* node.

1.  **VM Creation and Static IP Setup**:
    * Create your three virtual machines (Master, Worker1, Worker2) using your preferred hypervisor (VMware, VirtualBox, etc.). Ensure they meet the minimum resource recommendations (e.g., 4GB RAM, 40GB Storage, 4 vCPUs per VM for a demo).
    * Configure all VMs with a **"Bridged" network adapter** to allow them to communicate directly with your physical network and each other.
    * **Crucially**, follow the steps in this guide to install Ubuntu Server and configure a **static IP address** for each of your VMs:
        [https://github.com/Vonrisen/mtd-cluster/blob/main/ubuntu_installation_and_configuration.md](https://github.com/Vonrisen/mtd-cluster/blob/main/ubuntu_installation_and_configuration.md)
    * Ensure the Master node can connect to the Worker nodes via SSH using the username and password (or SSH keys) you will specify in the KubeKey configuration later.

2.  **Install Essential Packages**: On *each* of the three VMs (Master, Worker1, Worker2), install the following packages:
    * `conntrack`: Used by Kubernetes networking components (like kube-proxy) for connection tracking.
    * `socat`: A network utility often used for port forwarding and other communication tasks by Kubernetes.
    * `ebtables`: Manages Ethernet bridge filtering tables, needed for Kubernetes networking.
    * `ipset`: Used for managing IP sets, often utilized in network policy enforcement.

    Execute the following command on **each** VM:
    ```bash
    sudo apt update
    sudo apt upgrade -y
    sudo apt install -y socat conntrack ebtables ipset
    ```

At this point, all your VMs should be set up with static IPs, essential packages. You are ready to proceed with the KubeSphere installation.

---

## 2. Installing KubeSphere with KubeKey

KubeKey (`kk`) is a powerful command-line tool used to install Kubernetes clusters and KubeSphere. You will perform the following steps from the **Master node**.

1.  **Install Kubernetes Cluster with KubeKey**
    If you are accessing GitHub/Googleapis from a restricted location, please log in to any cluster node and run the following command to set the download region:
    ```bash
    export KKZONE=cn
    ```
    Run the following command to download the latest version of KubeKey:
    ```bash
    curl -sfL https://get-kk.kubesphere.io | sh -
    ```
    After the download is complete, a KubeKey binary file `kk` will be generated in the current directory.
    NoteIf the cluster node used to perform the operations cannot connect to the internet, you can manually download KubeKey on a device with internet access and then transfer it to the cluster node.
    Add execute permission to the KubeKey binary file `kk`:
    ```bash
    sudo chmod +x kk
    ```
    Create the installation configuration file `config-sample.yaml`:
    ```bash
    ./kk create config --with-kubernetes v1.33.0
    ```

2.  **Edit the Cluster Configuration File**: Open the generated configuration file using a text editor (like `nano`):
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

3.  **Create the Cluster**: Now, initiate the installation process using KubeKey and the modified configuration file:
    ```bash
    ./kk create cluster -f config-sample.yaml --with-local-storage
    ```
    This command will start the deployment. KubeKey will connect to each node via SSH, install Kubernetes components. This process can take some time depending on your network speed and VM resources.

4.  **Install Helm and KubeSphere Core**
    After the Kubernetes cluster installation completes, install **Helm** on the **Master node** by following the official documentation: [https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/)

    Once Helm is installed, run the following command on the cluster node (Master node) to install KubeSphere Core:
    ```bash
    # If you are accessing charts.kubesphere.io from a restricted location, replace charts.kubesphere.io with charts.kubesphere.com.cn
    helm upgrade --install -n kubesphere-system --create-namespace ks-core https://charts.kubesphere.io/main/ks-core-1.1.4.tgz --debug --wait
    ```
    **Note**: If you are accessing Docker Hub from a restricted location, add the following configuration after the above command to modify the default image pull address.
    ```bash
    --set global.imageRegistry=swr.cn-southwest-2.myhuaweicloud.com/ks
    --set extension.imageRegistry=swr.cn-southwest-2.myhuaweicloud.com/ks
    ```

---

## Conclusion

If you see the following information, it means that ks-core installation is successful:
   ```
	NOTES:
	Thank you for choosing KubeSphere Helm Chart.
	
	Please be patient and wait for several seconds for the KubeSphere deployment to complete.
	
	1. Wait for Deployment Completion
	
		Confirm that all KubeSphere components are running by executing the following command:
	
		kubectl get pods -n kubesphere-system
	
	2. Access the KubeSphere Console
	
		Once the deployment is complete, you can access the KubeSphere console using the following URL:
	
		http://192.168.6.10:30880
	
	3. Login to KubeSphere Console
	
		Use the following credentials to log in:
	
		Account: admin
		Password: P@88w0rd
	
	NOTE: It is highly recommended to change the default password immediately after the first login.
	
	For additional information and details, please visit https://kubesphere.io.
   ```

You have successfully installed a Kubernetes cluster with KubeSphere on your VMs using KubeKey. You can now explore the KubeSphere console to deploy and manage your banking application demo, monitor resources, manage users, and more.

**Remember that the allocated resources are suitable for a demo; production environments would require significantly more planning and resources.**
