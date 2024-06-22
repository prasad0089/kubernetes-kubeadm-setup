# Kubernetes Kubeadm Setup

This guide will help you set up a Kubernetes cluster using `kubeadm`, `kubelet`, and `kubectl`. Follow along with each section to complete the installation.

## Table of Contents

1. [Installing containerd](#installing-containerd)
2. [Configuring containerd](#configuring-containerd)
3. [Installing Kubernetes tools (kubeadm, kubelet, kubectl)](#installing-kubernetes-tools-kubeadm-kubelet-kubectl)
4. [Disabling swap](#disabling-swap)
5. [Enabling required kernel modules and sysctl settings](#enabling-required-kernel-modules-and-sysctl-settings)
6. [Initializing the Kubernetes cluster](#initializing-the-kubernetes-cluster)
7. [Setting up kubectl](#setting-up-kubectl)
8. [Installing Flannel CNI plugin](#installing-flannel-cni-plugin)
9. [Joining other nodes to the cluster](#joining-other-nodes-to-the-cluster)
10. [Using the control plane as a node for deployments](#using-the-control-plane-as-a-node-for-deployments)

## Installing containerd

1. Update the package list on your system:
    ```bash
    sudo apt-get update
    ```

2. Install necessary packages:
    ```bash
    sudo apt-get install -y apt-transport-https ca-certificates curl gpg
    ```

3. Download the GPG key for the Kubernetes repository and save it as a dearmored GPG key file:
    ```bash
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    ```

4. Add the Kubernetes repository to the system's package sources:
    ```bash
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    ```

5. Update the package list again to include the new repository:
    ```bash
    sudo apt-get update
    ```

6. Install containerd:
    ```bash
    sudo apt-get install -y containerd
    ```

[Back to Top](#kubernetes-kubeadm-setup-tutorial)

## Configuring containerd

1. Create the containerd configuration directory:
    ```bash
    sudo mkdir -p /etc/containerd
    ```

2. Generate a default configuration file for containerd and save it:
    ```bash
    containerd config default | sudo tee /etc/containerd/config.toml
    ```

3. Modify the containerd configuration to use systemd for cgroup management:
    ```bash
    sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
    ```

4. Restart the containerd service to apply the new configuration:
    ```bash
    sudo systemctl restart containerd
    ```

[Back to Top](#kubernetes-kubeadm-setup-tutorial)

## Installing Kubernetes tools (kubeadm, kubelet, kubectl)

1. Update package list:
    ```bash
    sudo apt-get update
    ```

2. Install kubelet, kubeadm, and kubectl:
    ```bash
    sudo apt-get install -y kubelet kubeadm kubectl
    ```

3. Mark these packages as "held" to prevent automatic updates:
    ```bash
    sudo apt-mark hold kubelet kubeadm kubectl
    ```

[Back to Top](#kubernetes-kubeadm-setup-tutorial)

## Disabling swap

1. Disable swap immediately:
    ```bash
    sudo swapoff -a
    ```

2. Comment out any swap entries in /etc/fstab to prevent swap from being re-enabled on reboot:
    ```bash
    sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
    ```

[Back to Top](#kubernetes-kubeadm-setup-tutorial)

## Enabling required kernel modules and sysctl settings

1. Create a configuration file to load the overlay and br_netfilter kernel modules at boot:
    ```bash
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF
    ```

2. Load these modules immediately:
    ```bash
    sudo modprobe overlay
    sudo modprobe br_netfilter
    ```

3. Create a sysctl configuration file for Kubernetes networking requirements:
    ```bash
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-iptables  = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward                 = 1
    EOF
    ```

4. Apply the sysctl settings immediately:
    ```bash
    sudo sysctl --system
    ```

[Back to Top](#kubernetes-kubeadm-setup-tutorial)

## Initializing the Kubernetes cluster

1. Initialize the Kubernetes control-plane, specifying the pod network CIDR for Flannel:
    ```bash
    sudo kubeadm init --pod-network-cidr=10.244.0.0/16
    ```

[Back to Top](#kubernetes-kubeadm-setup-tutorial)

## Setting up kubectl

1. Create the kubectl configuration directory:
    ```bash
    mkdir -p $HOME/.kube
    ```

2. Copy the admin kubeconfig file to the user's home directory:
    ```bash
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    ```

3. Change ownership of the config file to the current user:
    ```bash
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

[Back to Top](#kubernetes-kubeadm-setup-tutorial)

## Installing Flannel CNI plugin

1. Apply the Flannel manifest to the cluster, setting up the Container Network Interface:
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
    ```

[Back to Top](#kubernetes-kubeadm-setup-tutorial)

## Joining other nodes to the cluster

1. When you run `kubeadm init` on the control plane node, it outputs a join command at the end. It looks something like this:
    ```bash
    kubeadm join 192.168.1.100:6443 --token abcdef.1234567890abcdef \
        --discovery-token-ca-cert-hash sha256:1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
    ```

2. To join a new node to the cluster:
    - Install containerd, kubeadm, kubelet, and kubectl on the new node, following steps 1-5 from the previous instructions.
    - Run the join command on the new node:
        ```bash
        sudo kubeadm join 192.168.1.100:6443 --token abcdef.1234567890abcdef \
            --discovery-token-ca-cert-hash sha256:1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
        ```

3. If you've lost the join command or the token has expired, you can generate a new one on the control plane node:
    ```bash
    kubeadm token create --print-join-command
    ```

[Back to Top](#kubernetes-kubeadm-setup-tutorial)

## Using the control plane as a node for deployments

By default, the control plane node is not scheduled for regular workloads. This is a security measure to keep the control plane isolated. However, for smaller clusters or test environments, you might want to use the control plane for running workloads.

1. To allow scheduling on the control plane node:
    ```bash
    kubectl taint nodes --all node-role.kubernetes.io/control-plane-
    ```

2. Verify that the taint has been removed:
    ```bash
    kubectl describe node <control-plane-node-name> | grep Taints
    ```

3. If the taint was successfully removed, this should return no results or show other taints if any exist.

4. (Optional) If you later want to prevent workloads from being scheduled on the control plane again, you can re-add the taint:
    ```bash
    kubectl taint nodes <control-plane-node-name> node-role.kubernetes.io/control-plane:NoSchedule
    ```

[Back to Top](#kubernetes-kubeadm-setup-tutorial)

## License

This project is licensed under the MIT License. See the LICENSE file for details.
