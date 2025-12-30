# Overview

This home lab is designed for understanding the foundations of a k8s cluster using Kubeadm and other cloud-agnostic tools.

### Handy Links for Documentation

[Markdown Syntax Basics](https://www.markdownguide.org/basic-syntax/)

<br>

# VM setup

This lab uses Kubeadm to setup a control plane and worker nodes. To best emulate practical implementations, a 3 node control plane will be configured using Ubuntu.

My lab consists of a MacBook, so I will be using UTM as my virtualization platform and ARM based VMs.

> [!note]
> You can find control plane specs for kubeadm here:<br>
> [Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

> [!warning]
> All VMs created here are not bridged to any real networks-- they are all internal VM-only networks. These VMs are running directly on my Macbook and are not routable from the rest of my network, but they can reach out to the internet to download resources as needed. This is intentional. Any testing of the network can only be done from my laptop.

> [!error]
> My original configuration had 2 GB of RAM per control node. This worked until I installed Calico which ran into OOM isues, so I increased it.

## Control Plane Nodes

> [!note]
> LTS Version of Ubuntu Server (ARM) used:<br>
> [Ubuntu Server LTS Download](https://ubuntu.com/download/server/arm)

### Node Information

| Node       | Role    | IP Address     | CPUs | Memory | Disk  | OS             |
| ---------- | ------- | -------------- | ---- | ------ | ----- | -------------- |
| cp-node-01 | control | 192.168.64.201 | 2    | 4 GB   | 20 GB | Ubuntu 24.04.3 |
| cp-node-02 | control | 192.168.64.202 | 2    | 4 GB   | 20 GB | Ubuntu 24.04.3 |
| cp-node-03 | control | 192.168.64.203 | 2    | 4 GB   | 20 GB | Ubuntu 24.04.3 |

### Network Information

| Name     | Description | Network | Gateway |
| -------- | ----------- | ------- | ------- |
| Node     | The shared virtual network with my Mac (NATted to my home network). From the cluster's perspective, this is "external" | 192.168.64.0/24 | 192.168.64.1 |
| Pod      | Non-routable network used for pod networking       | 10.255.0.0/16 | N/A |
| Service  | Non-routable network used for services (ClusterIP) | 10.96.0.0/12 | N/A |

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
  - Your server's name: `cp-node-xx`
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

# Update the YAML file in /etc/netplan
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

Enable IP forwarding so that kubeadm doesn't complain.

```bash
/etc/sysctl.conf

# Uncomment line 28
net.ipv4.ip_forward=1
```

Load the new settings

```bash
sudo sysctl -p
```

We will be using a shared name for the API server configuration. This allows us to load balance the API server across the control plane nodes and can eventually be load balanced. Update the hosts file to use the API server's name and point it to the local host.

```bash
/etc/hosts
...
127.0.0.1    kube-api
```

<br>

> [!tip]
> Once the initial setup was complete and I validated SSH, I powered off the VMs and removed the display adapter. Because of how UTM works on Mac, each VM generates a window when it runs and closing the window causes the VM to pause. This was annoying, so I removed the display adapter.

<br>

## Kubeadm Setup (v1.35)

### Disable Swap

The default behavior of the kubelet is to fail if swap behavior occurs. So, we can disable swap on all nodes to prevent issues. Do this by modifying the `/etc/fstab` and commenting out the line for swap.

```bash
/etc/fstab

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

  # Set the version of the pause image. Kubeadm will warn of any version mismatches.
  # line 67
  [plugins."io.containerd.grpc.v1.cri"]
    ...
    sandbox_image = "registry.k8s.io/pause:3.10.1"

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
> The following configuration is only done on the first control plane node (**cp-node-01**). Configuring the other nodes will be done later on.

We will start by creating the cluster on the first node. Some notes on the installation document about this particular setup:
- We are using the IP address configured on the control plane nodes. No additional IPs and no fancy network setups. This appears to be recommended against anyways.
- We will use Calico for the CNI plugin.
- Since we plan on deploying a multi-node control plane, we will specify the `--control-plane-endpoint` argument and specify a DNS name of `kube-api`. We will need to create a hosts file entry to ensure this name resolves to the local ip address.
- We specify the IP range for the CNI plugin using `--pod-network-cidr`
- Kubeadm should automatically detect that we are using containerd, but I put in the `--cri-socket` parameter anyway. This just tells it to use containerd.
- I am using containerd version 1.7. When I run the command I get a warning message about needing to update the containerd version to one that suports RuntimeConfig in the future (version 2.0+ of containerd). It will be required in the next release of k8s. I am ignoring this for now.

Here is the command and the arguments we will use to set up the cluster:

```bash
sudo kubeadm init --control-plane-endpoint kube-api --pod-network-cidr 10.255.0.0/16 --cri-socket unix:///var/run/containerd/containerd.sock
```

<br>

> [!Tip]
> If you run into any errors using `kubeadm init`, run the `kubeadm reset` command to revert, fix the issues, then try again.

<br>

When the command runs successfully, you will get output that describes how to join additional nodes. Keep this output as it will come in handy later!

```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join kube-api:6443 --token xmwmwu.qakochmw8co1bc88 \
	--discovery-token-ca-cert-hash sha256:0b530721865f80641bf7fc24e8f63f409f7736c58b0d9352d67366e0b3249c59 \
	--control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join kube-api:6443 --token xmwmwu.qakochmw8co1bc88 \
	--discovery-token-ca-cert-hash sha256:0b530721865f80641bf7fc24e8f63f409f7736c58b0d9352d67366e0b3249c59 
```

### Cluster access

Let's verify the connectivity of our new cluster. We will be using the admin tools exclusively from **cp-node-01**, though you can certainly configure them on all 3 nodes if you'd like or even a separate jumpbox entirely.

Setup the config file so that we can have the context for `kubectl` to access the cluster:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify your access by running the following command and checking the health of the control plane pods:

```bash
kubectl get pods -A
```

Setting up some aliases to make things easier

```bash
alias k='kubectl'
alias kg='k get'
alias kgn='k get nodes'
alias kgp='k get pods'
alias kgs='k get services'
alias kgd='k get deployments'
alias kgpa='k get pods --all-namespaces'
alias kds='k describe service'
alias kdp='k describe pod'
alias klogs='k logs -f'
```

> [!note]
> The `coredns` pods will stay in a pending state until we install a CNI plugin. Everything else should be running.

To make things a bit easier to work with, we will go ahead and generate an SSH key pair, configure VSCode with a remote SSH connection, and download this repo to the remote server for easy access to our manifests.

Generate an SSH keypair on the **Mac**. Press enter to get through the prompts (no passphrase needed):

```bash
# By default, this saves the key ~/.ssh/id_ed25519
ssh-keygen -t ed25519

# Copy the key to the control plane servers
# If ssh-copy-id isn't available, you can manually copy the public key information to the ~/.ssh/authorized_keys of the target servers
ssh-copy-id jesse@192.168.64.201
ssh-copy-id jesse@192.168.64.202
ssh-copy-id jesse@192.168.64.203
```

> [!tip]
> Use the Remote - SSH extension to setup a remote connection in VSCode from your computer to the control node. This will make it a lot easier to manage the manifest files!

<br>

### Installing CNI Plugin (Calico)

Before we can add new nodes to the mix, we need a CNI plugin to handle our pod routing.

[Pod network configuration](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network)

[Calico setup](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises)

We will use the recommended approach of installing the Tigera Operator and custom resource definitions. The yaml files were downloaded from the calico documentation provided in the link above.

```bash
kubectl create -f ./calico/operator-crds.yaml
kubectl create -f ./calico/tigera-operator.yaml
```

We have the option to use either eBPF or the traditional iptables dataplane. eBPF is the newer version of the dataplane and has better performance. So, we will be using eBPF.
[Why move from iptables to eBPF](https://isovalent.com/blog/post/why-replace-iptables-with-ebpf/)

> [!warning]
> This resource file has been updated to use the pod network ranges specified in this document. (10.255.0.0/16 and a blocksize of /24 per node).

<br>

```bash
kubectl create -f ./calico/custom-resources-bpf.yaml

# Monitor the deployment. This might take a few minutes.
watch kubectl get tigerastatus

# During this, the api server may become unavailble and lead to connection issues.
```

<br>

> [!caution]
> It turned out that the `calico-apiserver` was crashing from OOM (`kubectl get pod -A`). I ended up increasing the memory of the node to 4 GB.

<br>

### Install K9s and MetricServer

Let's make life a little easier with useful metrics.

```bash
kubectl apply -f ./metrics-server/components.yaml
```

And now K9s for easy management. It's available through snap.

```bash
sudo snap install k9s --channel=latest/stable

# A symlink is needed to run it
sudo ln -s /snap/k9s/current/bin/k9s /snap/bin/k9s
```
<br>

> [!caution]
> The k9s snap didn't work. Looking into it...


## TODO
- Metrics server!
- Prometheus
- Static CoreDNS entries (for things like NFS server)
- Istio
- NFS Server
- Worker node
- Image registry
- Secret storage
- Encrypt etcd
- Velero backups

## Additional Tools
These tools provide additional quality of life improvements and troubleshooting assistance.

- podman
- crictl
- etcdtl
- nerdctl
- k9s
- kubens
- git