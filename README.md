# kube-v6
Instructions on how to instantiate a multi-node, IPv6-only Kubernetes cluster using the CNI bridge plugin and Host-local IPAM plugin for developing or exploring IPv6 on Kubernetes.

# Overview
So you'd like to take Kubernetes IPv6 for a test drive, or perhaps do some Kubernetes IPv6 development? The instructions below describe how to bring up a multi-node, IPv6-only Kubernetes cluster that uses the CNI bridge and host-local IPAM plugins, using kubeadm to stand up the cluster.

There have been many recent changes that have been added or proposed to Kubernetes for supporting IPv6 are either not merged yet, or they were merged after the latest official release of Kubernetesi (1.8.0). In the meantime, we need a way of exercising these yet "in-flight" IPv6 changes on a Kubernetes cluster. This wiki offers you two ways to include these changes in a Kubernetes cluster instance:

 * Using "canned", or precompiled binaries and container images for Kubernetes components
 * Compiling your own Kubernetes binaries and container images.

For instructional purposes, the steps below assume the topology shown in the following diagram, but certainly various topologies can be supported (e.g. using baremetal nodes or different IPv6 addressing schemes) with slight variations in the steps:

![Screenshot](kubernetes_ipv6_topology.png)

# FAQs

#### Why Use the CNI Bridge Plugin? Isn't it intended for single-node clusters?
The Container Networking Interface (CNI) [Release v0.6.0](https://github.com/containernetworking/plugins/releases/tag/v0.6.0) included support for IPv6 operation for the Bridge plugin and Host-local IPAM plugin. It was therefore considered a good reference plugin with which to test IPv6 changes that were being made to Kubernetes. Although the bridge plugin is intended for single-node operation (the bridge on each minion node is isolated), a multi-node cluster using the bridge plugin
can be instantiated using a couple of manual steps:

 * Provide each node with its unique pod address space (e.g. each node gets a unique /64 subnet for pod addresses).
 * Add static routes on each minion to other minions' pod subnets using the target minion's node address as a next hop.

#### Why run in IPv6-only mode rather than running in dual-stack mode?
The first phase of implementation for IPv6 on Kubernetes will target support for IPv6-only clusters. The main reason for this is that Kubernetes currently only supports/recognizes a single IP address per pod (i.e. no multiple-IP support). So even though the CNI bridge plugin supports dual-stack (as well as support for multiple IPv6 addresses on each pod interface) operation on the pods, Kubernetes will currently only be aware of one IP address per pod.

#### What is the purpose of NAT64 and DNS64 in the IPv6 Kubernetes cluster topology diagram?
We need to be able to test IPv6-only Kubernetes cluster configurations. However, there are many external services (e.g. DockerHub) that only work with IPv4. In order to interoperate between our IPv6-only cluster and these external IPv4 services, we configure a node outside of the cluster to host a NAT64 server and a DNS64 server. The NAT64 server operates in dual-stack mode, and it serves as a stateful translator between the internal IPv6-only network and the external IPv4 Internet. The DNS64 server synthesizes AAAA records for any IPv4-only host/server outside of the cluster, using a special prefix of 64:ff9b::. Any packets that are forwarded to the NAT64 server with this special prefix are subjected to stateful address translation to an IPv4 address/port.

#### Should I use global (GUA) or private (ULA) IPv6 addresses on the Kubernetes nodes?
You can use GUA, ULA, or a combination of both for addresses on your Kubernetes nodes. Using GUA addresses (that are routed to your cluster) gives you the flexibility of connecting directly to Kubernetes services from outside the cluster (e.g. by defining Kubernetes services using nodePorts or externalIPs). On the other hand, the ULA addresses that you choose can be convenient and predictable, and that can greatly simplify the addition of static routes between nodes and pods.

# Preparing the Nodes

## Set up node IP addresses
For the example topology show above, the eth2 addresses would be configured via IPv6 SLAAC, and the eth1 addresses would be statically configured as follows:
```
       Node        IP Address
   -------------   ----------
   NAT64/DNS64     fd00::64
   Kube Master     fd00::100
   Kube Minion 1   fd00::101
   Kube Minion 1   fd00::102
```

## (For convenience) Configure /etc/hosts on each node with the new addresses
Here's an example /etc/hosts file:
```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
fd00::64 kube-nat64-dns64
fd00::100 kube-master
fd00::101 kube-minion-1
fd00::102 kube-minion-2
```

## Add static routes between nodes, pods, and Kubernetes services
In the list of static routes below, the subnets/addresses used are as follows:
```
   Subnet/Address          Description
   --------------    ---------------------------
   64:ff9b::/96      Prefix used inside the cluster for packets requiring NAT64 translation
   fd00::101         Kube Minion 1
   fd00::102         Kube Minion 2
   fd00:101::/64     Kube Minion 1's pod subnet
   fd00:102::/64     Kube Minion 2's pod subnet
   fd00:1234::/64    Cluster's Service subnet
```

#### Static Routes on NAT64/DNS64 Server
Example: CentOS 7, entries in /etc/sysconfig/network-scripts/route6-eth1:
```
fd00:101::/64 via fd00::101 metric 1024
fd00:102::/64 via fd00::102 metric 1024
```

#### Static Routes on Kube Master
Example: CentOS 7, entries in /etc/sysconfig/network-scripts/route6-eth1:
```
64:ff9b::/96 via fd00::64 metric 1024
fd00:101::/64 via fd00::101 metric 1024
fd00:102::/64 via fd00::102 metric 1024
```

#### Static Routes on Kube Minion 1
Example: CentOS 7, entries in /etc/sysconfig/network-scripts/route6-eth1:
```
64:ff9b::/64 via fd00::64 metric 1024
fd00:102::/64 via fd00::102 metric 1024
fd00:1234::/64 via fd00::100 metric 1024
```

#### Static Routes on Kube Minion 2
Example: CentOS 7, entries in /etc/sysconfig/network-scripts/route6-eth1:
```
64:ff9b::/64 via fd00::64 metric 1024
fd00:101::/64 via fd00::101 metric 1024
fd00:1234::/64 via fd00::100 metric 1024
```

## Set sysctl settings for forwarding and using iptables/ip6tables
For example, on CentOS 7 hosts, add the following to /etc/sysctl.conf:
```
sudo -i
cat << EOT >> /etc/sysctl.conf
net.ipv6.conf.all.forwarding=1
net.bridge.bridge-nf-call-ip6tables=1
EOT
sudo sysctl -p /etc/sysctl.conf
exit
```

## Configure and install NAT64 and DNS64 on the NAT64/DNS64 server
For installing on a Ubuntu host, refer to the [NAT64-DNS64-UBUNTU-INSTALL.md](NAT64-DNS64-UBUNTU-INSTALL.md) file.

For installing on a CentOS 7 host, refer to the [NAT64-DNS64-CENTOS-INSTALL.md](NAT64-DNS64-CENTOS-INSTALL.md) file.

## If Using VirtualBox VMs as Kubernetes Nodes: Delete the Default IPv4 Route on All Kubernetes Nodes
VirtualBox typically sets up a VM's eth0 interface as an IPv4 NAT port to the external world, and configures the interface using DHCP. The presence of an IPv4 address (typically 10.0.2.15) on a VM's eth0 in an otherwise IPv6-only Kubernetes node won't interfere with IPv6-only operation of the cluster. However, another part of the DHCP configuration is an IPv4 default route. This IPv4 default route can interfere with control channel communication in an IPv6-only cluster because Kubernetes favors any IPv4 address that is on a interface that is associated with an IPv4 default route over any IPv6 addresses when choosing a node IP for that node. The node IP is what is used for inter-node control plane communication, so this needs to be an IPv6 address for IPv6-only operation.

So if you are using VirtualBox VMs as Kubernetes nodes, you should delete the default IPv4 route, e.g.:
```
ip route delete default via 10.0.2.2 dev eth0
```
Since DHCP reconfiguration happens periodically (IP lease expiry), you may also want to disable DHCP configuration on the VM's eth0.

# Install Standard (Upstream) Kubernetes Packages
On the Kubernetes master and minion nodes, install docker, kubernetes, and kubernetes-CNI.

#### For CentOS 7 / Fedora based hosts
````
sudo -i
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://yum.kubernetes.io/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
       https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
exit

sudo -i
setenforce 0
yum install -y docker kubelet kubeadm kubectl kubernetes-cni
systemctl enable docker && systemctl start docker
systemctl enable kubelet && systemctl start kubelet
exit
```

The kubelet is now restarting every few seconds, as it waits in a crashloop for kubeadm to tell it what to do.

