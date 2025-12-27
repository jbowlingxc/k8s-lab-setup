# Overview

This home lab is designed for understanding the foundations of a k8s cluster using Kubeadm and other cloud-agnostic tools.

### Handy Links for Documentation

[Markdown Syntax Basics](https://www.markdownguide.org/basic-syntax/)

<br>

# VM setup

This lab uses Kubeadm to setup a control plane and worker nodes. To best emulate practical implementations, a 3 node control plane will be configured using Ubuntu.

My lab consists of a Macbook, so I will be using UTM as my virtualization platform and ARM based VMs.

>[!note]
> You can find control plane specs for kubeadm here:<br>
> [Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

>[!warning]
> All VMs created here are not bridged to any real networks-- they are all internal VM-only networks. These VMs are running directly on my Macbook and are not routable from the rest of my network, but they can reach out to the internet to download resources as needed. This is intentional. Any testing of the network can only be done from my laptop.

## Control Plane Nodes

> [!note]
> LTS Version of Ubuntu Server (ARM) used:<br>
> [Ubuntu Server LTS Download](https://ubuntu.com/download/server/arm)

### Node Information

| Node       | Role    | IP Address     | CPUs | Memory | Disk  | OS             |
| ---------- | ------- | -------------- | ---- | ------ | ----- | -------------- |
| cp-node-01 | control | 192.168.64.201 | 2    | 2 GB   | 20 GB | Ubuntu 24.04.3 |
| cp-node-02 | control | 192.168.64.202 | 2    | 2 GB   | 20 GB | Ubuntu 24.04.3 |
| cp-node-03 | control | 192.168.64.203 | 2    | 2 GB   | 20 GB | Ubuntu 24.04.3 |

### OS Installation

The following installation options were selected during the Ubuntu Server setup:
- "Choose the base for the installation" -> Ubuntu Server
- Accept the DHCP network configuration (we can change later)
- No Proxy address
- Keep default mirror address
- Use an entire disk (keep default here)
- Accept default partitioning layout
- Profile configuration
  - Your name: `jesse`
  - Your servers name: `cp-node-xx`
  - Pick a username: `jesse`
  - Choose a password: `***********`
  - Confirm your password: `***********`
- Skip Ubuntu Pro
- Select the box to install the SSH Server, then select Next
- Skip the additional packages and finish the installation

### Network Configuration

After successful installation and ejecting the installation media, I configured a static IP address on each node using the shared network range that my Macbook provided to my VM network. These commands were ran on each node:

```bash
# Update the timezone
sudo timedatectl set-timezone America/Los_Angeles

# Get current IP information
ip a

# Get the default gateway
ip route

# Update the yaml file in /etc/netplan
sudo vim /etc/netplan/50-cloud-init.yaml
```

Set the static IP

```yaml
network:
  version: 2
  ethernets:
    enp0s1:
      addresses:
        [192.168.64.201/24] # .202, .203 for the other nodes
      routes:
        - to: default
          via: 192.168.64.1
          metric: 100
      nameservers:
        search: [local]
        addresses: [8.8.8.8, 4.2.2.2]
      dhcp6: false
      dhcp4: false
```

Apply the new configuration

```bash
sudo netplan apply
```

<br>

> [!tip]
> Once the initial setup was complete and I validated SSH, I powered off the VMs and removed the display adapter. Because of how UTM works on Mac, each VM genereates a window when it runs and closing the window causes the VM to pause. This was annoying, so I removed the display adapter.

<br>

## Kubeadm Setup (v1.35)

### Disable Swap

The default behavior of the kubelet is to fail if swap behavior occurs. So, we can disable swap on all nodes to prevent issues. Do this by modifying the `/etc/fstab' and commenting out the line for swap.

```bash
sudo vim /etc/fstab

# /swap.img       none    swap    sw      0       0
```

Restart and validate.
```bash
sudo systemctl reboot
# Wait for reboot... then log back in
swapon --show
# If there is no output, then we have successfully disabled swap
```

### Install containerd

This is not installed by kubeadm, so you must install it manually.

```bash
sudo apt-get update
sudo apt-get install -y containerd
```

Generate a default configuration and update the cgroup to use systemd, as recommended by the Kubernetes documentation. I had to create the `/etc/containerd` folder first.

```bash
sudo mkdir /etc/containerd
containerd config default > /etc/containerd/config.toml
```

These instructions helped with configuring the driver for containerd. Make sure to verify the version of containerd you are running: `containerd --version`<br>
[Configuring the systemd cgroup driver](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd)

```toml
/etc/containerd/config.toml
  # Only changed values are listed here
  # line 128
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    ...
    SystemdCgroup = true
```

Restart the containerd service

```bash
sudo systemctl restart containerd

# Validate
containerd config dump | grep SystemdCgroup
```

### Install kube packages

This requires the kube repository. Each version of Kubernetes has its own repository, so use the right one for the target version.

```bash
# Install prereqs
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Perform another update of the repo packages and install kubelet, kubeadm, and kubectl
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# Freeze the versions
sudo apt-mark hold kubelet kubeadm kubectl

# Enable the kubelet service before starting the kubeadm install
sudo systemctl enable --now kubelet

# The kubelet process will continue to crash in a loop until kubeadm is configured
sudo systemctl status kubelet
```

<br>

### Create the cluster
We are referencing the following document:<br>
[Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

> [!warning]
> This part is coming soon.

## Additional Tools
These tools provide additional quality of life improvements and troubleshooting assistance.

- podman
- crictl
- etcdtl
- nerdctl
- k9s
- kubens
- git