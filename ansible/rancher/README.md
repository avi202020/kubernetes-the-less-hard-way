# Configure

Copy then edit gateway, domain...

```
cp group_vars/all.yml.template group_vars/all.yml
```

# RancherOS
RancherOS is the OS which will run Kubernetes Cluster.

On each worker|controller nodes
- Boot RancherOS iso
- Configure network  
To configure static ip
```
sudo ros config set rancher.network.interfaces.eth0.address X.X.X.X/24
sudo ros config set rancher.network.interfaces.eth0.gateway X.X.X.Y
sudo ros config set rancher.network.interfaces.eth0.dhcp false
sudo system-docker restart network
```
**Ansible**
To be able to run ansible on nodes, it needs python.

- Change to ubuntu console with
```
sudo ros console switch ubuntu -f
sudo apt update
sudo apt install python -y
```
- Change default ``rancher`` password
```
sudo passwd rancher
```
Then

```
ansible-playbook 00-configure.yml -k
# enter rancher password
# it will fail with unreachable status du to reboot, that's normal
```

# RKE
RKE automates the setup process of Kubernetes cluster.

Generate RKE config based on ansible hosts file.
```
ansible-playbook 01-local.yml
```
Then run rke manually with
```
cd rke
./rke up
```
Test with
```
# Install kubeconfig
cp kube_config_cluster.yml ~/.kube/config
chmod 600 ~/.kube/config
kubectl get nodes
```

# Rancher
Rancher is the tool to manage Kubernetes Clusters on a WebUI.
Helm is the Kubernetes package manager.

Install helm package manager on your host and add rancher repository
```
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
# Create a namespace
kubectl create namespace cattle-system
```
[Install cert-manager](https://rancher.com/docs/rancher/v2.x/en/installation/install-rancher-on-k8s/#5-install-cert-manager), then
```
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.my.org
```