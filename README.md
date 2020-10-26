IN PROGRESS

# Kubernetes The (Less) Hard Way

Ansible playbooks to learn how to host Kubernetes on premise.
Kind of between [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) and [KubeSpray](https://github.com/kubernetes-sigs/kubespray).

## VMs provisionning

| Hostname        | OS                  |
|-----------------|---------------------|
| k8s-controller1 | Ubuntu Server 20.04 |
| k8s-controller2 | Ubuntu Server 20.04 |
| k8s-controller3 | Ubuntu Server 20.04 |
| k8s-worker1     | Ubuntu Server 20.04 |
| k8s-worker2     | Ubuntu Server 20.04 |
| k8s-worker3     | Ubuntu Server 20.04 |
| k8s-haproxy     | Debian 10           |

## Configuration

Create dirs and copy configuration template files.
```
mkdir -p ssl/csr kubeconfigs
cp ansible/hosts.template ansible/hosts
cp ansible/group_vars/all.yml.template ansible/group_vars/all.yml
```
Add all hostnames in ``ansible/hosts``.
Configure vars in ``ansible/group_vars/all.yml``.
- Domain
- Cluster Name

## Network and DNS

All hosts needs an private IP on the same subnet.
Create DNS or ``hosts`` file entries for each VM.

## Firewall

All trafic between VMs should not be filtered.
To access services from outside, you should open in your firewall:

| Service        | Port     | Destination |
|----------------|----------|-------------|
| kube-apiserver | 6443/tcp | k8s-haproxy |

## Run Ansible playbooks

Install SSH, authorize your SSH public key, then test if VMs are reachable.
```
ansible all -m ping
```

Please read playbooks before running.
Prepare environment with
```
ansible-playbook 00-configure.yml
```

**SSL** : Generate Certificate Authority, Certificates, and kubeconfigs.
```
ansible-playbook 01-local.yml
```

**Kubernetes Control Plane and etcd** : Install and configure Kubernetes controllers.
```
ansible-playbook 02-controllers.yml
```
Test etcd with
```
ansible-playbook tests.yml -v -t etcd
```
Returns on each
```
dd5995046894cdd4, started, k8s-controller3, https://X.X.X.X:2380, https://X.X.X.X:2379, false
ec778b58d61b4683, started, k8s-controller1, https://X.X.X.X:2380, https://X.X.X.X:2379, false
f1c47e23a339d1cf, started, k8s-controller2, https://X.X.X.X:2380, https://X.X.X.X:2379, false
```

Test control plane with
```
ansible-playbook tests.yml -v -t components
```
Returns on each
```
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-1               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}   

Test health with
```
ansible-playbook tests.yml -v -t healthz
```
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Fri, 16 Oct 2020 16:57:04 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 2
Connection: keep-alive
Cache-Control: no-cache, private
X-Content-Type-Options: nosniffA

ok
```

**HAProxy** : Install and configure HA proxy in front of controllers.

```
ansible-galaxy install manala.haproxy -p roles
ansible-playbook 03-haproxy.yml
ansible haproxy -m shell -a "systemctl restart haproxy"
```
Test with
```
ansible-playbook tests.yml -v -t haproxy
```
Returns
```
{
  "major": "1",
  "minor": "18",
  "gitVersion": "v1.18.6",
  "gitCommit": "dff82dc0de47299ab66c83c626e08b245ab19037",
  "gitTreeState": "clean",
  "buildDate": "2020-07-15T16:51:04Z",
  "goVersion": "go1.13.9",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

**Kubernetes Worker Nodes** : Install and configure Kubernetes worker nodes.
By default, it uses ``containerd`` runtime, and ``cni`` network addon.
You can change these by uncommenting specific lines in the playbook.
```
ansible-playbook 04-workers.yml
```
Test with
```
ansible-playbook tests.yml -v -t nodes
```
Returns on each
```
NAME                 STATUS   ROLES    AGE     VERSION
k8s-worker1   Ready    <none>   2m27s   v1.18.6
k8s-worker2   Ready    <none>   2m17s   v1.18.6
k8s-worker3   Ready    <none>   2m17s   v1.18.6
```

**Remote Access** : Configure kubectl on your host

Test with
```
ansible-playbook tests.yml -v -t remote
```

**Network** : Add routes between pod subnets
```
ansible-playbook 05-network.yml
```

Test with
```
ansible-playbook tests.yml -v -t route
```
Returns on each
```
# Where X are pod subnets, and Y default interface of worker
default via $gateway dev ens3 proto static 
X.X.1.0/24 via Y.Y.Y.1 dev ens3 
X.X.2.0/24 dev cnio0 proto kernel scope link src X.X.2.1 linkdown 
X.X.3.0/24 via Y.Y.Y.3 dev ens3
Y.Y.Y.Y/24 dev ens3 proto kernel scope link src Y.Y.Y.2
```

**CoreDNS** : Deploy DNS cluster Add-On
```
ansible-playbook 06-addons.yml -t coredns
```

Test with
```
kubectl get pods -l k8s-app=kube-dns -n kube-system
```
Returns
```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-XXXXXXXXXX-XXXXX   1/1     Running   0          105s
coredns-XXXXXXXXXX-XXXXX   1/1     Running   0          105s
```
Then see [Kubernetes The Hard Way #Dns Addon verification](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/12-dns-addon.md#verification)

**Metrics** : Deploy metrics server
```
ansible-playbook 06-addons.yml -t metrics
```
Test with
```
kubectl top nodes
```

**Dashboard** : Deploy Webui
```
ansible-playbook 06-addons.yml -t dashboard
kubectl proxy
```
Access at [http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/)and connect with ``token``.
