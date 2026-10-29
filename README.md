
# Ubuntu Server Setup for Kubernetes 1.34

This guide describes how to prepare an Ubuntu server for Kubernetes using a static IP address, containerd, and Kubernetes 1.34 packages hosted on an internal Nexus repository.

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Configure Static IP](#2-configure-static-ip)
3. [Disable Cloud-Init Network Management](#3-disable-cloud-init-network-management)
4. [Set Hostname](#4-set-hostname)
5. [Configure DNS](#5-configure-dns)
6. [Configure Ubuntu APT Repositories](#6-configure-ubuntu-apt-repositories)
7. [Configure Timezone](#7-configure-timezone)
8. [Install Containerd](#8-install-containerd)
9. [Configure Containerd](#9-configure-containerd)
10. [Disable Swap](#10-disable-swap)
11. [Active IP Forwarding](#11-Active-IP-Forwarding)
12. [Configure Kubernetes Repository](#11-configure-kubernetes-repository)
13. [Install Kubernetes Components](#12-install-kubernetes-components)
14. [Verify Installation](#13-verify-installation)

## 1. Prerequisites

- Ubuntu Server 22.04 LTS (Jammy).
- Network adapter configured in bridged mode if running in a virtual machine.
- Access to the internal Nexus repository.
- A user with sudo privileges.
- A static IP address reserved for this server.

### Example server configuration

| Setting | Value |
|---|---|
| Hostname | `master01` |
| IP address | `192.168.100.2/24` |
| Gateway | `192.168.100.1` |
| DNS | `185.51.200.2`, `178.22.122.100` |
| Timezone | `Asia/Tehran` |
| Kubernetes repository | Internal Nexus |
| Kubernetes version | 1.34 |

> **Important:** Replace the example IP addresses and interface name with values appropriate for your VLAN. Do not assign an IP address already used by another device.

## 2. Configure Static IP

### 2.1 Identify the network interface

```bash
vi /etc/netplan/00-installer-config.yaml
```

Confirm the interface name. The following example uses `ens33`.

### 2.2 Back up the existing configuration

```bash
sudo cp /etc/netplan/00-installer-config.yaml \
  /etc/netplan/00-installer-config.yaml.bak
```

### 2.3 Edit Netplan configuration

```bash
sudo vi /etc/netplan/00-installer-config.yaml
```

Example configuration:

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.100.2/24
      routes:
        - to: default
          via: 192.168.100.1
      nameservers:
        addresses:
          - 185.51.200.2
          - 178.22.122.100
```

Replace `ens33` with the actual interface name.

### 2.4 Apply the configuration

Validate the configuration:

```bash
sudo netplan generate
```

If connected remotely, test the configuration before committing it:

```bash
sudo netplan try
```

Otherwise, apply it directly:

```bash
sudo netplan apply
```

Verify connectivity:

```bash
ip addr
ip route
ping -c 4 192.168.100.1
ping -c 4 185.51.200.2
```

## 3. Disable Cloud-Init Network Management

Only disable cloud-init if this server does not need it for provisioning or networking.

```bash
sudo systemctl stop cloud-init
sudo systemctl disable cloud-init
sudo systemctl mask cloud-init
```

Disabling cloud-init is not necessary on every Ubuntu installation. If the goal is only to stop cloud-init from managing network configuration, use the appropriate cloud-init network configuration instead.

## 4. Set Hostname

Set the hostname to `master01`:

```bash
sudo hostnamectl set-hostname master01
echo "192.168.100.2 master01" >> /etc/hosts
```

Start a new shell:

```bash
exec bash
```

Verify:

```bash
hostnamectl
hostname
```

## 5. Configure DNS SHEKAN

DNS is configured through Netplan in Section 2.

Check the effective resolver configuration:

```bash
resolvectl status
cat /etc/resolv.conf
vim /etc/resolv.conf
```

```text
nameserver 185.51.200.2
nameserver 178.22.122.100
```

If your environment requires specific DNS servers, confirm that they are reachable and that the intended resolver configuration is active.

Avoid manually editing `/etc/resolv.conf` unless your environment specifically requires it; Ubuntu may regenerate this file.

## 6. Configure Ubuntu APT Repositories

### 6.1 Check Ubuntu version

```bash
lsb_release -a
```

The repository configuration below is for Ubuntu 22.04 LTS (Jammy).

### 6.2 Back up the current configuration

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
```

### 6.3 Edit the repository list

```bash
sudo vi /etc/apt/sources.list
```

Configure the internal Nexus mirror:

```text
deb http://nexus.serverops.ir/repository/apt-proxy/ jammy main restricted
deb http://nexus.serverops.ir/repository/apt-proxy/ jammy-updates main restricted

deb http://nexus.serverops.ir/repository/apt-proxy/ jammy universe
deb http://nexus.serverops.ir/repository/apt-proxy/ jammy-updates universe

deb http://nexus.serverops.ir/repository/apt-proxy/ jammy multiverse
deb http://nexus.serverops.ir/repository/apt-proxy/ jammy-updates multiverse

deb http://nexus.serverops.ir/repository/apt-proxy/ jammy-backports main restricted universe multiverse

deb http://nexus.serverops.ir/repository/apt-proxy-security/ jammy-security main restricted
deb http://nexus.serverops.ir/repository/apt-proxy-security/ jammy-security universe
deb http://nexus.serverops.ir/repository/apt-proxy-security/ jammy-security multiverse
```

> Use HTTPS if supported by your internal Nexus server. Verify repository availability and signing configuration before proceeding.

Update package indexes:

```bash
sudo apt update
```

## 7. Configure Timezone

Set the timezone:

```bash
sudo timedatectl set-timezone Asia/Tehran
```

Verify:

```bash
timedatectl
date
```

## 8. Install Containerd

Install containerd:

```bash
sudo apt install -y containerd
```

Enable and start the service:

```bash
sudo systemctl enable --now containerd.service
```

Check service status:

```bash
sudo systemctl status containerd.service
```

## 9. Configure Containerd

### 9.1 Generate the default configuration

Create the configuration directory:

```bash
sudo mkdir -p /etc/containerd
```

Back up an existing configuration, if present:

```bash
sudo cp /etc/containerd/config.toml \
  /etc/containerd/config.toml.bak
```

If the file does not yet exist, generate it:

```bash
sudo containerd config default | \
  sudo tee /etc/containerd/config.toml > /dev/null
```

### 9.2 Configure the systemd cgroup driver

Edit the configuration:

```bash
sudo vi /etc/containerd/config.toml
```

For containerd 1.x, locate the `runc` runtime configuration under the CRI plugin and set:

```toml
SystemdCgroup = true
```

For containerd 2.x, locate the equivalent `runc` runtime configuration under the CRI runtime plugin and set:

```toml
SystemdCgroup = true
```

Ensure the setting is in the correct `runc` runtime section. Do not add duplicate sections or change unrelated settings blindly.

If the generated configuration contains:

```toml
disabled_plugins = ["cri"]
```

remove `cri` from the disabled plugin list; Kubernetes requires a working CRI runtime.

Restart containerd:

```bash
sudo systemctl restart containerd.service
```

Verify:

```bash
sudo systemctl status containerd.service
sudo ctr version
sudo crictl info
```

The last command requires `crictl` to be installed and configured for the containerd socket.

## 10. Disable Swap

Temporarily disable swap:

```bash
sudo swapoff -a
```

Inspect active swap:

```bash
swapon --show
free -h
```

If swap is configured in `/etc/fstab`, back up the file:

```bash
sudo cp /etc/fstab /etc/fstab.bak
```

Edit it:

```bash
sudo vi /etc/fstab
```

Comment out the applicable swap entry by placing `#` at the beginning of that line.

Verify again:

```bash
swapon --show
```

Swap should be disabled for the standard Kubernetes configuration used in this guide.

## 11. Active ip forwarding

```bash
cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
echo br_netfilter > /etc/modules-load.d/k8s.conf
sysctl --system
```


## 12. Configure Kubernetes Repository

### 12.1 Install required packages

```bash
sudo apt install -y ca-certificates curl gpg
```

### 12.2 Create the keyring directory

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

### 12.3 Download the repository signing key

Verify the key URL and fingerprint with your repository administrator before trusting it.

```bash
curl -fsSL \
  http://nexus.serverops.ir/repository/key-hosted/key-hosted/Release.key \
  | sudo gpg --dearmor --yes \
      -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Set permissions:

```bash
sudo chmod 0644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

### 12.4 Add the Kubernetes repository

Create the repository list:

```bash
sudo vi /etc/apt/sources.list.d/kubernetes.list
```

Add:

```text
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://nexus.serverops.ir/repository/apt-kubernetes-1-34/ /
```

Update package indexes:

```bash
sudo apt update
```

If `apt update` reports a signature, certificate, or repository error, resolve it before installing Kubernetes packages.

## 13. Install Kubernetes Components

Install the required components:

```bash
sudo apt install -y kubelet kubeadm kubectl
```

Check available package versions:

```bash
apt list -a kubeadm
apt list -a kubelet
apt list -a kubectl
```

Check installed versions:

```bash
kubeadm version
kubectl version --client
kubelet --version
```

For a reproducible cluster, select compatible versions and install or pin exact package versions according to your organization's repository policy.

Enable kubelet:

```bash
sudo systemctl enable kubelet
```

The kubelet may restart repeatedly until the node has been initialized or joined to a cluster. This can be expected before cluster initialization.

## 14. Verify Installation

Run the following checks:

```bash
hostnamectl
ip addr
ip route
resolvectl status
timedatectl
swapon --show
sudo systemctl is-active containerd
sudo systemctl is-enabled containerd
kubeadm version
kubectl version --client
kubelet --version
```

Confirm that:

- The hostname is `master01`.
- The server has the intended static IP.
- The gateway and DNS servers are reachable.
- The timezone is correct.
- Swap is disabled.
- Containerd is running with the correct cgroup configuration.
- The Kubernetes packages are installed at compatible versions.

## Next Steps

This guide prepares the Ubuntu server and installs Kubernetes components. It does not initialize the cluster.

For a control-plane node, the next step is to configure the cluster network CIDR and run `kubeadm init` with values appropriate for the selected CNI plugin and network.

For a worker node, initialize the control-plane node first, then use the generated `kubeadm join` command.

Do not run `kubeadm init` or `kubeadm join` until the node's networking, container runtime, and Kubernetes versions have been verified.
