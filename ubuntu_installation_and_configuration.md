# Guide to Installing and Configuring a Linux VM with a Static IP

This guide walks you through the steps to install a Linux virtual machine (using a standard ISO image) and configure it with a static IP address. This is particularly useful for servers or development environments where a consistent IP is required for accessibility.

## 1. ISO Download and Initial Setup

1.  **Download the ISO**: Visit the [official webpage](https://ubuntu.com/download/server) to download the Ubuntu Server ISO.
2.  **Create the Virtual Machine**: Use an hypervisor like VMware Workstation, VirtualBox, or any other of your preference to create a new virtual machine.
3.  **Installation Process**: Boot the VM using the downloaded ISO image. During the installation process:
    * Leave most settings as default unless specific requirements dictate otherwise.
    * **Important**: If your installer offers the option (like the Ubuntu Server installer), select "Install OpenSSH Server". This will allow you to connect to the VM remotely via SSH immediately after installation.
4.  **Verify and Install SSH**: After the installation is complete and you have logged into the VM's console:
    * **Check if SSH is Installed**: First, let's check if the OpenSSH server package was installed during setup.
        ```bash
        dpkg -s openssh-server
        ```
        If the package is installed, this command will show its status and version. If it's *not* installed, it will indicate the package is not found.
    * **Check if SSH Service is Running**: If the package is installed, check if the SSH service is active.
        ```bash
        systemctl status ssh
        ```
        Look for "Active: active (running)".
    * **Install SSH (if needed)**: If `dpkg -s openssh-server` indicated the package was not installed:
        ```bash
        sudo apt update
        sudo apt install openssh-server
        ```
    * **Enable and Start SSH (if needed)**: If `systemctl status ssh` showed the service as inactive or if you just installed it:
        ```bash
        sudo systemctl enable ssh # Ensures SSH starts on boot
        sudo systemctl start ssh # Starts the SSH service now
        systemctl status ssh # Verify it's running
        ```
    * **Update the System**: Now that SSH is potentially running (allowing you to connect remotely), perform a full system update:
        ```bash
        sudo apt update
        sudo apt upgrade
        ```
    * A reboot might be necessary after installing SSH or performing major system upgrades.

## 2. Configuring a Static IP

To ensure your VM's IP address doesn't change each time it reboots, you should set a static IP. We will use Netplan, the default network configuration system in many recent Linux distributions like Ubuntu 18.04+.

1.  **Identify Your Network Interface, Current IP, and Gateway**:
    * Open a terminal in your VM.
    * Run `ip a` to see your network interfaces and their current IP addresses. Look for the interface name (e.g., `eth0`, `ens33`, `enp0s3`) that has an IP address assigned (usually the one connected to your network). Note down its name and the current IP (like `192.168.1.50/24`).
    * Run `ip r` to see your routing table. The line starting with `default` shows the gateway IP address and the interface used for it. Note down the gateway IP (e.g., `192.168.1.1`).
    * *Tip*: While `ip a` provides a lot of detail, you can sometimes get a cleaner view of just IPv4 addresses with `ip -4 a show scope global` or look specifically at the default route with `ip r`. However, `ip a` is the most comprehensive way to see interface names.
2.  **Find the Netplan Configuration File**: Netplan configuration files are located in `/etc/netplan/`. The filename often relates to `cloud-init` or the installation source.
    ```bash
    ls /etc/netplan/
    ```
    Identify the `.yaml` file (e.g., `50-cloud-init.yaml`).
3.  **Edit the Netplan Configuration File**: Open the identified `.yaml` file for editing. Replace `<filename>` with the actual name you found.
    ```bash
    sudo nano /etc/netplan/<filename>
    ```
4.  **Configure the Static IP**: Modify the file content to match the following structure. **Replace the placeholder values enclosed in `< >`** with the information you gathered in Step 1 and the desired static IP address for your VM.

    ```yaml
    network:
      version: 2
      ethernets:
        <network_interface>: # e.g., ens33 or eth0
          addresses:
            - <VM_Address>/24 # e.g., 192.168.1.100/24 (the /24 indicates the subnet mask 255.255.255.0)
          routes:
            - to: default
              via: <GatewayIP> # e.g., 192.168.1.1 (the IP of your router or gateway)
          nameservers:
            addresses: [8.8.8.8, 8.8.4.4] # Google DNS servers, you can use others if you prefer
          dhcp4: false # <<< Crucial: Disable DHCP to use the static configuration below
    ```
    * `<network_interface>`: The name of your network interface (e.g., `ens33`).
    * `<VM_Address>/24`: The static IP address you want to assign to the VM, followed by the CIDR notation for your subnet mask (`/24` is common for `255.255.255.0`). Choose an IP address that is *not* currently used and is within the same network as your Gateway.
    * `<GatewayIP>`: The IP address of your router or network gateway (e.g., `192.168.1.1`).
    * `addresses: [8.8.8.8, 8.8.4.4]`: IP addresses of DNS servers (Google DNS in this example). You can use your ISP's DNS, internal DNS servers, or other public DNS servers.
    * `dhcp4: false`: This line is essential. It disables the automatic assignment of an IP address via DHCP and tells Netplan to use the static configuration you've defined.

5.  **Apply the Configuration**: Save the changes to the file (In `nano`, press `Ctrl + X`, then `Y`, then `Enter`). Then, execute the command to apply the new network configuration:
    ```bash
    sudo netplan apply
    ```
6.  **Verify**: Run `ip a` again to verify that your VM now has the static IP address you configured. You should also be able to ping your gateway (`ping <GatewayIP>`) and a public website (`ping google.com`) to test connectivity.

Your virtual machine is now configured with a static IP address and is ready for use.
