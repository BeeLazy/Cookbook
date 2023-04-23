# How to install Charmed Kubernetes

## About
This document describes how to install Charmed Kubernetes on Ubuntu Server 22.04.  

## Installation

### Update Ubuntu hosts
First we'll make sure our Ubuntu hosts are up to date
```console
sudo apt-get update
sudo apt-get dist-upgrade
```

### DNS
If you dont have working dns, add all the nodes to the hosts file on all nodes.
```console
sudo nano /etc/hosts
```

### Enable firewall
Enable ufw
```console
sudo ufw allow ssh
sudo ufw enable
```

### Install LXD
[LXD](https://linuxcontainers.org/) is a system container and VM hypervisor that allows you to create a local cloud. It can be installed via snap.  
If you dont have snap command, see [Installing snapd](https://snapcraft.io/docs/installing-snapd)  

LXD should already be installed. If it is not, it can be added with:
```console
sudo snap install lxd --classic
```

### Grant accees to lxd group
Add our user to the lxd group. It should already be added.
```console
sudo adduser $USER lxd
newgrp lxd
```

### Initialize LXD
LXD provides an interactive dialogue to configure your local cloud during the initialization procedure
```console
lxd init
```

The init script itself may vary depending on the version of LXD. You can use most default options in the dialogue. The important configuration options for Charmed Kubernetes are:  
- LXD clustering: Enable if installation is to be clustered
- Storage Pool: Use the 'dir' storage type

```console
beelazy@charmedjuju1:~$ lxd init
Would you like to use LXD clustering? (yes/no) [default=no]: yes
What IP address or DNS name should be used to reach this node? [default=192.168.0.228]: charmedjuju1
Are you joining an existing cluster? (yes/no) [default=no]: no
What name should be used to identify this node in the cluster? [default=charmedjuju1]: charmedjuju1
Setup password authentication on the cluster? (yes/no) [default=no]: no
Do you want to configure a new local storage pool? (yes/no) [default=yes]: yes
Name of the storage backend to use (btrfs, dir, lvm, zfs) [default=zfs]: dir
Do you want to configure a new remote storage pool? (yes/no) [default=no]: no
Would you like to connect to a MAAS server? (yes/no) [default=no]: no
Would you like to configure LXD to use an existing bridge or host interface? (yes/no) [default=no]: no
Would you like to create a new Fan overlay network? (yes/no) [default=yes]: yes
What subnet should be used as the Fan underlay? [default=auto]: auto
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]: yes
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]: yes
config:
  core.https_address: charmedjuju1:8443
networks:
- config:
    bridge.mode: fan
    fan.underlay_subnet: auto
  description: ""
  name: lxdfan0
  type: ""
  project: default
storage_pools:
- config: {}
  description: ""
  name: local
  driver: dir
profiles:
- config: {}
  description: ""
  devices:
    eth0:
      name: eth0
      network: lxdfan0
      type: nic
    root:
      path: /
      pool: local
      type: disk
  name: default
projects: []
cluster:
  server_name: charmedjuju1
  enabled: true
  member_config: []
  cluster_address: ""
  cluster_certificate: ""
  server_address: ""
  cluster_password: ""
  cluster_certificate_path: ""
  cluster_token: ""
```

### Set firewall rules
Set ufw firewall rules
```console
sudo ufw allow in on lxdfan0
sudo ufw route allow in on lxdfan0
sudo ufw route allow out on lxdfan0
sudo ufw allow 8443
```

### Add more nodes
To add more nodes to the cluster, we will need to generate join tokens
```console
lxc cluster add charmedjuju2
lxc cluster add charmedjuju3
```

Then on each of the nodes, run **sudo lxd init** to add them to the cluster
```console
sudo lxd init
```

> :warning: **Caution:** On nodes joining the cluster, the lxd command has to be run with **sudo**.  

Check syslog if there are connection problems between nodes
```console
cat /var/log/syslog
```

### Test installation
When all nodes are added to the cluster, we're ready to test the installation
```console
beelazy@charmedjuju1:~$ sudo lxd cluster show
# Latest dqlite segment ID: 1024

members:
- id: 1
  name: charmedjuju1
  address: charmedjuju1:8443
  role: voter
- id: 2
  name: charmedjuju2
  address: charmedjuju2:8443
  role: voter
- id: 3
  name: charmedjuju3
  address: charmedjuju3:8443
  role: voter
```

```console
beelazy@charmedjuju1:~$ lxc cluster list
+--------------+---------------------------+-----------------+--------------+----------------+-------------+--------+-------------------+
|     NAME     |            URL            |      ROLES      | ARCHITECTURE | FAILURE DOMAIN | DESCRIPTION | STATE  |      MESSAGE      |
+--------------+---------------------------+-----------------+--------------+----------------+-------------+--------+-------------------+
| charmedjuju1 | https://charmedjuju1:8443 | database-leader | x86_64       | default        |             | ONLINE | Fully operational |
|              |                           | database        |              |                |             |        |                   |
+--------------+---------------------------+-----------------+--------------+----------------+-------------+--------+-------------------+
| charmedjuju2 | https://charmedjuju2:8443 | database        | x86_64       | default        |             | ONLINE | Fully operational |
+--------------+---------------------------+-----------------+--------------+----------------+-------------+--------+-------------------+
| charmedjuju3 | https://charmedjuju3:8443 | database        | x86_64       | default        |             | ONLINE | Fully operational |
+--------------+---------------------------+-----------------+--------------+----------------+-------------+--------+-------------------+
```

And finally, test a deployment
```console
lxc launch images:alpine/3.11 web1
lxc launch images:alpine/3.11 web2
lxc launch images:alpine/3.11 web3
lxc list
```

### Install Juju
[Juju](https://jaas.ai/) is a tool for deploying, configuring and operating complex software on public or private clouds. It can be installed with snap
```console
sudo snap install juju --classic
```

### Add juju controller
The Juju controller is used to manage the software deployed through Juju, from deployment to upgrades to day-two operations. 
One Juju controller can manage multiple projects or workspaces, which in Juju are known as ‘models’.  

Juju comes preconfigured to work with LXD. A cloud created by using LXD containers on the local machine is known as localhost to Juju. 
To begin, you need to create a Juju controller for this cloud:
```console
juju bootstrap localhost
```

### Add a Kubernetes model
The model holds a specific deployment. It is a good idea to create a new one specifically for each deployment. 
```console
juju add-model k8s
```

Remember that you can have multiple models on each controller, so you can deploy multiple Kubernetes clusters or other applications.

### Deploy Kubernetes
Deploy the Kubernetes bundle to the model. This will add instances to the model and deploy the required applications. 
This can take up to 20 minutes depending on your machine. 
```console
juju deploy charmed-kubernetes
```

### Monitor the deployment
Juju is now busy creating instances, installing software and connecting the different parts of the cluster together, which can take several minutes. 
You can monitor what’s going on by running:
```console
watch -c juju status --color
```

To view the last twenty log messages for the 'k8s' model:
```console
juju debug-log -m k8s -n 20
```

### Install and configure kubectl
You will need kubectl to be able to use your Kubernetes cluster. If it is not already installed, it is easy to add via a snap package:
```console
sudo snap install kubectl --classic
```

The config file for accessing the newly deployed cluster is stored in the cluster itself and will be available as soon as the installation has settled. 
You should use the following command to retrieve it (create a .kube directory if it was not created after kubectl installation):
```console
juju scp kubernetes-control-plane/0:config ~/.kube/config
```

> :warning: **Caution:** If you have multiple clusters you will need to manage the config file rather than just replacing it. 
See the [Kubernetes documentation](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) for more information on managing multiple clusters.

### Verify Kubernetes install
You can verify that kubectl is configured correctly and can see the cluster by running:
```console
kubectl cluster-info
```

Now you can run pods inside the Kubernetes cluster:
```console
kubectl create -f example.yaml
```

List all pods in the cluster:
```console
kubectl get pods
```

List all services in the cluster:
```console
kubectl get services
```

### Access the Kubernetes dashboard
```console
kubectl proxy
```

The URL for the dashboard will then be  
[http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/)

![Kubernetes Dashboard](../../img/charmedk8s-1node.png "Kubernetes Dashboard")

## Related links
[Installing snapd - snapcraft.io](https://snapcraft.io/docs/installing-snapd)  
[Install Kubernetes - ubuntu.com](https://ubuntu.com/kubernetes/install)  
[Install troubleshooting guide - ubuntu.com](https://ubuntu.com/kubernetes/docs/install-local#troubleshooting)  
[Creating a LXD cluster - acloudguru.com](https://acloudguru.com/hands-on-labs/creating-a-lxd-cluster)  
