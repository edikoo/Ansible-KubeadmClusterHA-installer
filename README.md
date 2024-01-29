# Ansible Playbooks to install an High Available (HA) Kubernetes (multi-master) cluster using Kubeadm, Containerd, HAproxy, Keepalived and Internal ETCD

# Prerequisites:
- Install Ansible and a forward proxy on the Ansible host
  - Ansible:
    for macos `brew install ansible`
    for linux `yum install ansible`


## Structure

- `roles/helm-setup`: Contains tasks, for setting up Helm.
- `roles/haproxy`: Contains tasks, templates, and handlers for setting up HAProxy.
- `roles/keepalived`: Contains tasks, templates, and handlers for setting up Keepalived.
- `roles/commons`: Contains common tasks such as setting the hostname, installing dependencies, disabling swap, and setting the timezone.
- `roles/configure-kernel`: Configures kernel parameters and modules required by Kubernetes, including loading `br_netfilter` and `overlay` modules.
- `roles/containerd`: Installs and configures `containerd` as the container runtime and sets up Kubernetes components (`kubelet`, `kubeadm`, `kubectl`).
- `roles/cluster-setup`: Initializes the Kubernetes cluster using `kubeadm` and sets up the required configuration for the control plane and worker nodes.
- `roles/join-masters`: Adds additional control plane nodes to the Kubernetes cluster.
- `roles/join-workers`: Adds worker nodes to the Kubernetes cluster.
- `haproxy_setup.yml`: Main playbook to apply the HAProxy and Keepalived configuration.
- `cluster_setup.yml`: Main playbook for setting up the Kubernetes cluster using kubeadm.
- `helm_setup.yml`: Main playbook for setting up Helm.

# Environment preparation:

Specify the Master and Workers in the `inventory/hosts.yaml` file:
```
[master] # these are all the master nodes
[worker] # these are all the worker nodes
[haproxy] # these are all the haproxy nodes
```

# HAProxy and Keepalived Setup

This project sets up HAProxy and optionally Keepalived on specified hosts using Ansible.


`inventory/group_vars/haproxy.yml`: Define variables like `keepalived_enabled: true or false` and if you enable keepalived also define variable `keepalived_virtual_ip`.

`inventory/group_vars/haproxy.yml` example if you want to DISABLE keepalived: 
```yaml
keepalived_enabled: false
```


`inventory/group_vars/haproxy.yml` example if you want to ENABLE keepalived: 
```yaml
keepalived_enabled: true
keepalived_virtual_ip: 10.80.24.143
```

#### When Keepalived enabled

if you enable Keepalived, you must provide specific variables for each HAProxy host to ensure proper configuration and operation. These variables include:

- `keepalived_interface` : The network interface Keepalived uses for sending and receiving VRRP packets.
- `keepalived_state` : Defines the initial state of Keepalived (MASTER or BACKUP). The MASTER initially owns the virtual IP.
- `keepalived_priority` : Determines the priority for the VRRP instance. Higher values denote higher priority. In case of a failure, the backup with the highest priority will take over.

`inventory/hosts.yaml` example:
```yaml
haproxy:
  hosts:
    haproxy-1:
      # ... other vars ...
      keepalived_interface: eth1
      keepalived_state: MASTER
      keepalived_priority: 100
    haproxy-2:
      # ... other vars ...
      keepalived_interface: eth1
      keepalived_state: BACKUP
      keepalived_priority: 99
```


Run the playbook using the following command:
```
ansible-playbook -i inventory/hosts.yaml haproxy_setup.yaml
```


# Install a highly available kubernetes using kubeadm


Update the `inventory/group_vars/all.yaml` section:
- setup the control plain endpoint (VIP address) E.G. Virtual Ip or If you don't have a virtual IP, use the Haproxy IP
- setup The pod network cidr
- setup CNI
- choose the desired versions for (kubernetes, kubelet, kubectl, kubeadm)

`inventory/group_vars/all.yaml` contains following varaibles:

- `control_plane_endpoint` : The address and port on which the Kubernetes API server is accessible. This can be an HAProxy IP address, a virtual IP address, or a init master node IP address. Format: <IP_ADDRESS>:<PORT>. Example: 10.80.24.143:6443.
- `pod_network_cidr` : The CIDR range for the pod network. This is used by the network plugin to allocate IP addresses to pods. Example: 192.168.0.0/16.
- `cni` : The URL for the Container Network Interface (CNI) manifest. This is used to install the network plugin in your cluster. Example: "https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml" for Calico.
- `kubernetes_version` : The version of Kubernetes to install. This will be used for the Kubernetes API server, controller manager, and scheduler. Example: "1.27.2".
- `kubelet_version` : The version of kubelet to install. Kubelet runs on all nodes in the cluster and performs tasks such as starting pods and containers. Example: "1.27.2-00"
- `kubelet_version` : The version of kubeadm to install. Kubeadm is used to initialize and manage your Kubernetes cluster. Example: "1.27.2-00".
- `kubectl_version` : The version of kubectl to install. Kubectl is a command-line tool for interacting with the Kubernetes cluster. Example: "1.27.2-00".
- `timezone` : Define this variable to ensure that all nodes in your cluster operate under the same timezone. This setting will be applied by the commons role. You can specify any valid timezone string (e.g., "Asia/Tbilisi", "America/New_York", "Europe/London").

Run the playbook using the following command:

```
ansible-playbook -i inventory/hosts.yaml cluster_setup.yaml
```


## Appendix


### if you use virtualbox and want to add new master node you need to specify --apiserver-advertise-address masterNodeIP example:
#### kubeadm join 10.80.24.141:6443 --token 9a45rv.6panndt4icqu6anu --discovery-token-ca-cert-hash sha256:c89dbd80e863a8d274cb32b2ffb4cdca5debf3e78c7caf821165ff7eea872fb6  --control-plane --certificate-key 666ae3f464595535043f20a1f0b0769117dd901ab88430ba70c1478d2efd58da --apiserver-advertise-address masterNodeIP
