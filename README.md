# Kubernetes role for Ansible

This role can be used to deploy a Kubernetes cluster with a fully automated and idempotent implementation of several components.

## Features

This role can be configured to enable all of these features:

- **Single or multi master cluster implementation** with HAProxy and Keepalived for High Availability.

- **Multi network add-ons** Flannel and Calico.

- Kubernetes **Dashboard**.

- **Users management** with certificate generation and `kubeconfig` file update.

- **Ceph-CSI** StorageClass for block devices.

- **MetalLB** load balancer for baremetal environments.

- **Ingress NGINX** for service exposition.

- **Cert Manager** for automated certificate management.

## Install the cluster with the Ansible Playbook

Check out the main [README](https://github.com/mmul-it/ansible/blob/master/README.md) for details on how to install the requirements, you'll typically use the role by launching the `kubernetes.yml` playbook, like this:

```
user@lab ~ # ansible-playbook \
-i $HOME/Work/Git/github.com/mmul-it/ansible/inventory/lab \
$HOME/Work/Git/github.com/mmul-it/ansible/kubernetes.yml
```

Note that you can chose anytime to reset everything by passing `k8s_reset` as `true`:

```
user@lab ~ # ansible-playbook \
-i $HOME/Work/Git/github.com/mmul-it/ansible/inventory/lab \
$HOME/Work/Git/github.com/mmul-it/ansible/kubernetes.yml \
-e k8s_reset=true
```

<u><strong>NOTE</strong></u>: this will reset your entire cluster, so use it with caution.

## Interact with the cluster after the installation

Once the playbook completes its execution the best way to interact with the cluster is by using the `kubectl` command that can be installed as follows:

```
user@lab ~ # curl -s -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

user@lab ~ # chmod +x kubectl

user@lab ~ # sudo mv kubectl /usr/local/bin
```

The Kubernetes role produces a local directory which contains the main kubeconfig file, named admin.conf. The easiest way to use it is by exporting the KUBECONFIG variable, like this:

```
user@lab ~ # export KUBECONFIG=~/kubernetes/admin.conf 
```

From now until the end of the session, every time you'll `kubectl` it will rely on the credentials contained in that file:

```
user@lab ~ # kubectl cluster-info
Kubernetes control plane is running at https://192.168.122.199:8443
CoreDNS is running at https://192.168.122.199:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

user@lab ~ # kubectl get nodes
NAME           STATUS   ROLES           AGE   VERSION
kubernetes-1   Ready    control-plane   26h   v1.25.3
kubernetes-2   Ready    control-plane   26h   v1.25.3
kubernetes-3   Ready    control-plane   26h   v1.25.3
kubernetes-4   Ready    <none>          26h   v1.25.3
```

It is also possible to use different users to log into the cluster, check the [Users](#Users) section for details.

## Configuration

### Inventory

A typical inventory depends on what you want to deploy, looking at the example `lab`  you can declare inside the hosts file (see [inventory/lab/hosts](https://github.com/mmul-it/ansible/blob/master/inventory/lab/hosts)) all the nodes:

```ini
# Kubernetes hosts
[k8s_nodes]
kubernetes-1 k8s_role=master run_non_infra_pods=true
kubernetes-2 k8s_role=master run_non_infra_pods=true
kubernetes-3 k8s_role=master run_non_infra_pods=true
kubernetes-4 k8s_role=worker
```

You'll set which nodes will act as master and also whether or not those will run non infrastructure pods (so to make the master also a worker).

Then you can define, inside group file (see [inventory/lab/group_vars/k8s_nodes.yml](https://github.com/mmul-it/ansible/blob/master/inventory/lab/group_vars/k8s_nodes.yml)), all the additional configurations, depending on what do you want to implement.

### Kubernetes cluster

If you want to implement a multi-master, high availability cluster you'll need to specify these variables:

```yaml
k8s_cluster_name: kubelab

k8s_master_node: kubernetes-1
k8s_master_port: 6443
k8s_master_cert_key: "91bded725a628a081d74888df8745172ed842fe30c7a3898b3c63ca98c7226fd"

k8s_multi_master: true
k8s_balancer_VIP: 192.168.122.199
k8s_balancer_interface: eth0
k8s_balancer_port: 8443
k8s_balancer_password: "d6e284576158b1"

k8s_wait_timeout: 1200

k8s_master_ports:
  - 2379-2380/tcp
  - 6443/tcp
  - 8443/tcp
  - 10250/tcp
  - 10257/tcp
  - 10259/tcp
```

This will bring up a cluster starting from node `kubernetes-1` enabling multi master via `k8s_multi_master` and setting the VIP address and the interface.

**<u>Note</u>**: you'll want to change both `k8s_master_cert_key` and `k8s_balancer_password` for better security.

**<u>Note</u>**: it is possible to have a more atomic way to configure pods network cidr, worker ports , nodeports ranges and firewall management, you can check [the defaults file](https://github.com/mmul-it/ansible/blob/master/roles/kubernetes/defaults/main.yml#L50-L67).

### Network Add-on

The Kubernetes role supports Flannel and Calico network add-ons. The configuration depends on which add-on you want to implement.

For Flannel you'll need something like:

```yaml
# Flannel addon
k8s_network_addon: flannel
k8s_network_addon_ports:
  - 8285/udp
  - 8472/udp
```

To check how to implement Calico have a look at [the defaults file](https://github.com/mmul-it/ansible/blob/master/roles/kubernetes/defaults/main.yml#L69-L75).

### Dashboard

The Kubernetes dashboard can be implemented by adding this to the configuration:

```yaml
k8s_dashboard_enable: true
```

Once the installation completes the easiest way to access the dashboard is by using the `kubectl proxy` and then access the related url: http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

A login prompt will be presented, and you can login by passing a token. By default the Kubernetes role creates a user named `dashboard-user` (you can override it).

To retrieve the token you'll need use `kubectl`, like this:

```
user@lab ~ # kubectl -n kubernetes-dashboard create token dashboard-user
<YOUR TOKEN>
```

Copy and paste the output of the above command inside the prompt and you'll complete the login.

### Users

It is possible to add users to your cluster, by declaring something like this:

```yaml
k8s_users:
  - name: pod-viewer
    namespace: default
    role_name: pod-viewer-role
    role_rules_apigroups: '""'
    role_rules_resources: '"pods","pods/exec","pods/log"'
    role_rules_verbs: '"*"'
    rolebinding_name: pod-viewer-rolebinding
    cert_expire_days: 3650
    update_kube_config: true
```

This will create a local directory containing these files:

```
user@lab ~ # ls -1 kubernetes/users/
pod-viewer.crt
pod-viewer.csr
pod-viewer.key
users.conf
users-rolebindings.yaml
users-roles.yaml
```

The `users.conf` file can then be used to access the cluster with this user, like this:

```
user@lab ~ # export KUBECONFIG=~/kubernetes/users/users.conf 

rasca@catastrofe [~]> kubectl config get-contexts 
CURRENT   NAME                       CLUSTER   AUTHINFO           NAMESPACE
*         kubernetes-admin@kubelab   kubelab   kubernetes-admin   
          pod-viewer@kubelab         kubelab   pod-viewer         default

user@lab ~ # kubectl config use-context pod-viewer@kubelab 
Switched to context "pod-viewer@kubelab".

user@lab ~ # kubectl config get-contexts 
CURRENT   NAME                       CLUSTER   AUTHINFO           NAMESPACE
          kubernetes-admin@kubelab   kubelab   kubernetes-admin   
*         pod-viewer@kubelab         kubelab   pod-viewer         default

user@lab ~ # kubectl get pods
No resources found in default namespace.
```

### Ceph CSI

The Kubernetes role actually supports the implementation of the Ceph CSI StorageClass. It can be defined as follows:

```yaml
k8s_ceph_csi_enable: true
k8s_ceph_csi_id: lab-ceph
k8s_ceph_csi_secret_userid: kubernetes
k8s_ceph_csi_secret_userkey: AQAWvU5jjBHSGhAAuAXtHFt0h05B5J/VHERGOA==
k8s_ceph_csi_clusterid: d046bbb0-4ee4-11ed-8f6f-525400f292ff
k8s_ceph_csi_pool: kubepool
k8s_ceph_csi_monitors:
  - 192.168.122.11:6789
  - 192.168.122.12:6789
  - 192.168.122.13:6789
```

Then it will be possible to declare new PVC:

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd-pvc
  namespace: rasca
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-rbd-sc
```

And related Pod:

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: csi-rbd-demo-pod
  namespace: rasca
spec:
  containers:
    - name: web-server
      image: nginx
      volumeMounts:
        - name: mypvc
          mountPath: /var/lib/www/html
  volumes:
    - name: mypvc
      persistentVolumeClaim:
        claimName: rbd-pvc
        readOnly: false
```

**<u>NOTE</u>**: at the moment only `rbd` provisioning is supported.

### MetalLB

To enable [MetalLB](https://metallb.universe.tf/) a load-balancer implementation for baremetal Kubernetes clusters, using standard routing protocols, it is sufficient to declare:

```yaml
k8s_metallb_enable: true
k8s_metallb_pools:
  - name: 'first-pool'
    addresses: '192.168.122.100-192.168.122.130'
```

Then it will be possible to use this LoadBalancer to create IPs inside the declared pool range (check next `ingress-nginx` example to understand how).

### Ingress NGINX

To enable [Ingress NGINX](https://github.com/kubernetes/ingress-nginx), an Ingress controller for Kubernetes using NGINX as a reverse proxy and load balancer, it is sufficient to declare:

```yaml
k8s_ingress_nginx_enable: true
k8s_ingress_nginx_services:
  - name: ingress-nginx-lb
    spec:
      type: LoadBalancer
      loadBalancerIP: 192.168.122.100
      ports:
      - name: port-1
        port: 80
        protocol: TCP
      - name: port-2
        port: 443
        protocol: TCP
```

This will install everything related to the controller, and assign the `loadBalancerIP` that is part of the range supplied by MetalLB, by exposing both `80` and `443` ports.

### Cert Manager

To enable [Cert Manager](https://cert-manager.io/), a controller to automate certificate management in Kubernetes, it is sufficient to declare:

```yaml
k8s_cert_manager_enable: true
k8s_cert_manager_issuers:
  - name: letsencrypt
    cluster: true
    acme:
      server: https://acme-v02.api.letsencrypt.org/directory
      email: rasca@mmul.it
      privateKeySecretRef:
        name: letsencrypt
      solvers:
      - http01:
          ingress:
            class: nginx
```

This will install everything related to the controller and create a cluster issuer that will use `letsencrypt` with `http01` challenge resolution, via NGINX ingress class.

Once everything is installed and you want to expose an application, you can test everything by using something like this yaml:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: rasca
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: index-html
  namespace: rasca
data:
  index.html: |
    This is my faboulous Webserver!
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: rasca
  labels:
    app: nginx
spec:
  containers:
    - name: web-server
      image: nginx
      volumeMounts:
      - name: docroot
        mountPath: /usr/share/nginx/html
  volumes:
    - name: docroot
      configMap:
        name: index-html
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: rasca
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
  name: nginx-ingress
  namespace: rasca
spec:
  ingressClassName: nginx
  rules:
  - host: nginx.apps.kubelab.mmul.it
    http:
      paths:
      - backend:
          service:
            name: nginx-service
            port:
              number: 80
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - nginx.apps.kubelab.mmul.it
    secretName: nginx.apps.kubelab.mmul.it
```

If you look specifically on the last resource, the `Ingress` named `nginx-ingress` you will see two important sections:

- Under `metadata` -> `annotations` the annotation`cert-manager.io/cluster-issuer: letsencrypt`

- Under `spec:` -> `tls` the host declaration.

With this in place, after some time, you'll have your cert served for the exposed service.
