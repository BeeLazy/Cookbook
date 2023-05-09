multipass launch --cloud-init MicroK8s-ArgoCD-2.7.1.yaml \
--timeout 1200 \
--name argolab271 \
--memory 12G \
--cpus 6 \
--disk 50G

```console
package_update: true
package_upgrade: true

packages:
- python3-pip

snap:
  commands:
  - snap install juju --classic
  - snap install microk8s --classic --channel=latest/stable
  - snap alias microk8s.kubectl kubectl
  - snap refresh

runcmd:
- export NEEDRESTART_MODE=a
- export DEBIAN_PRIORITY=critical
- DEBIAN_FRONTEND=noninteractive apt-get -y upgrade
- DEBIAN_FRONTEND=noninteractive apt-get -y autoremove

- |
  # Disable swap
  sysctl -w vm.swappiness=0
  echo "vm.swappiness = 0" | tee -a /etc/sysctl.conf
  swapoff -a

- |
  # Disable unnecessary services
  systemctl disable man-db.timer man-db.service --now
  systemctl disable apport.service apport-autoreport.service  --now
  systemctl disable apt-daily.service apt-daily.timer --now
  systemctl disable apt-daily-upgrade.service apt-daily-upgrade.timer --now
  systemctl disable unattended-upgrades.service --now
  systemctl disable motd-news.service motd-news.timer --now
  systemctl disable bluetooth.target --now
  systemctl disable ua-messaging.service ua-messaging.timer --now
  systemctl disable ua-timer.timer ua-timer.service --now
  systemctl disable systemd-tmpfiles-clean.timer --now

  # Disable IPv6
  # Disabling IPv6 at kernellevel causes kubectl to fail 
  #echo "net.ipv6.conf.all.disable_ipv6=1" | tee -a /etc/sysctl.conf
  #echo "net.ipv6.conf.default.disable_ipv6=1" | tee -a /etc/sysctl.conf
  #echo "net.ipv6.conf.lo.disable_ipv6=1" | tee -a /etc/sysctl.conf
  #sysctl -p

- |
  # Create juju directory if it dont exists
  sudo -u ubuntu mkdir -p /home/ubuntu/.local/share/juju

- |
  # Add the 'ubuntu' user to the Microk8s group:
  sudo usermod -a -G microk8s ubuntu

  # Give the 'ubuntu' user permissions to read the ~/.kube directory:
  sudo -u ubuntu mkdir -p /home/ubuntu/.kube
  sudo chown -f -R ubuntu /home/ubuntu/.kube

  # Create the 'microk8s' group:
  newgrp microk8s

  # Wait for ready
  microk8s status --wait-ready

  # Enable the necessary Microk8s addons:
  microk8s.enable hostpath-storage
  microk8s.enable dns

  # Wait for storage to become available
  microk8s.kubectl rollout status deployments/hostpath-provisioner -n kube-system -w --timeout=600s

  # Bootstrap
  juju bootstrap microk8s microk8s

  # dump config (this is needed for utils such as k9s or kdash)
  microk8s config | sudo -u ubuntu tee /home/ubuntu/.kube/config > /dev/null

  # Deploy ArgoCD
  microk8s.kubectl create namespace argocd
  microk8s.kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.7.1/manifests/install.yaml

  # Wait for argocd-server to be ready
  microk8s.kubectl rollout status service/argocd-server -n argocd -w --timeout=600s

final_message: "The system is finally up, after $UPTIME seconds"
```

https://github.com/apeskalle/charm-dev-utils
https://technekey.com/using-cloud-init-to-bootstrap-a-virtual-machine/
https://cloudinit.readthedocs.io/en/latest/reference/examples.html
https://github.com/canonical/cloud-init
