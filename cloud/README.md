# cloud
This section contains information about viritualization and cloud related technologies like:
- LXD
- Kubernetes

## Contents

### cloud-init
Contains cloud-init configurations

### docs
Contains documentation and guides

### examples
Contains example configurations used in the docs

### img
Contains images and gfx artifacts used in docs

## Guides

**Hyper-V**:
- [How-to install Hyper-V in Hyper-V](docs/en-US/HyperV-NestedViritualization.md)
  - How-to enable nested viritualization and setup networking so one can run Hyper-V inside a Hyper-V VM

**Multipass**:
- [How-to install Multipass](docs/en-US/Multipass-HowtoInstall.md)
  - How-to install Multipass, a tool to create Ubuntu VMs in Linux, Windows or Mac
- [How-to setup ArgoCD and MicroK8s on a laptop](docs/en-US/Multipass-HowtoSetupArgoCDinMicroK8sOnLaptop.md)
  - Use Multipass to install MicroK8s Kubernetes
  - Setup ArgoCD to pull a github repository
- [SSH Access](docs/en-US/Multipass-SSHAccess.md)
  - How-to generate SSH keypair and use it to connect to Multipass VMs
  
**MicroK8s**:
- [How-to install MicroK8s](docs/en-US/MicroK8s-HowtoInstall.md)
  - How-to install MicroK8s
- [How-to install a MicroK8s HA cluster](docs/en-US/MicroK8s-HowtoSetupMultinodeHighAvailabilityCluster.md)
  - Expand the MicroK8s installation with more nodes to create a HA cluster
- [How-to access MicroK8s services](docs/en-US/MicroK8s-AccessingServices.md)
  - Accessing services in Kubernetes from outside kubernetes
    - Access Kubernetes Dashboard with proxy
	- Access Kubernetes service with port-forward
	- Access Kubernetes service with NodePort
	- Access Kubernetes service with Ingress
	- Access clustered Kubernetes service with MetalLB
- [Setup certificate management](docs/en-US/MicroK8s-CertManager.md)
  - Use cert-manager to automatically manage Let's Encrypt certificates

**LXD**:
- [How-to install LXD](docs/en-US/LXD-HowtoInstall.md)
  - Install a 3 noded LXD cluster
  - Basic tasks like starting and stopping containers

**SSH**:
- [Security of Interactive and Automated Access Management Using Secure Shell (SSH)](docs/en-US/NIST.IR.7966.pdf)
  - National Institute of Standards and Technology reference doc IR.7966
