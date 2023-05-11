# cloud-init
This section contains [cloud-init](https://github.com/canonical/cloud-init) configurations.  

## MicroK8s

### MicroK8s
- snap: **juju** --classic
- snap: **microk8s** --classic --channel=latest/stable
- alias: microk8s.kubectl kubectl

### MicroK8s-ArgoCD-**X**.**Y**.**Z**
- snap: **juju** --classic
- snap: **microk8s** --classic --channel=latest/stable
- kubectl: **arcocd** v**X**.**Y**.**Z**
- alias: microk8s.kubectl kubectl

## Related links
[cloud-init - github.com](https://github.com/canonical/cloud-init)  
