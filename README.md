# ![Kubelab Ansible Role](./images/kubelab-github-header.png)

This role can be used to deploy a Kubernetes cluster with a fully automated and
idempotent implementation of several components.

[![Lint the project](https://github.com/mmul-it/kubelab/actions/workflows/main.yml/badge.svg)](https://github.com/mmul-it/kubelab/actions/workflows/main.yml)
[![Ansible Galaxy](https://img.shields.io/badge/ansible--galaxy-kubelab-blue.svg)](https://galaxy.ansible.com/mmul/kubelab)

## Features

This role can be configured to enable all of these features:

- **Single or multi control plane cluster implementation** with HAProxy and Keepalived
  for High Availability.

- **Multi network add-ons** Flannel and Calico.

- Kubernetes **Dashboard**.

- **Users management** with certificate generation and `kubeconfig` file update.

- **Ceph-CSI** StorageClass for block devices.

- **MetalLB** load balancer for baremetal environments.

- **Ingress NGINX** for service exposition.

- **Cert Manager** for automated certificate management.

## Install the cluster with the Ansible Playbook

The best way to prepare the environment is to use a Python VirtualEnv,
installing ansible using `pip3`:

```console
user@lab ~ # python3 -m venv ansible
user@lab ~ # source ansible/bin/activate
(ansible) user@lab ~ # pip3 install ansible
Collecting ansible
  Using cached ansible-7.5.0-py3-none-any.whl (43.6 MB)
...
...
Installing collected packages: resolvelib, PyYAML, pycparser, packaging, MarkupSafe, jinja2, cffi, cryptography, ansible-core, ansible
Successfully installed MarkupSafe-2.1.2 PyYAML-6.0 ansible-7.5.0 ansible-core-2.14.5 cffi-1.15.1 cryptography-40.0.2 jinja2-3.1.2 packaging-23.1 pycparser-2.21 resolvelib-0.8.1
```

Then you will need this role, and in this case using `ansible-galaxy` is a good
choice to make it all automatic:

```console
(ansible) user@lab ~ # ansible-galaxy install mmul.kubelab -p ansible/roles/
Starting galaxy role install process
- downloading role 'kubelab', owned by mmul
- downloading role from https://github.com/mmul-it/kubelab/archive/main.tar.gz
- extracting mmul.kubelab to /home/rasca/ansible/roles/mmul.kubelab
- mmul.kubelab (main) was installed successfully
```

With the role in place you can complete the requirements, again by using `pip3`:

```console
(ansible) user@lab ~ # pip3 install -r ansible/roles/mmul.kubelab/requirements.txt
...
...
Successfully installed ansible-vault-2.1.0 cachetools-5.3.0 certifi-2023.5.7 charset-normalizer-3.1.0 google-auth-2.18.0 idna-3.4 kubernetes-26.1.0 oauthlib-3.2.2 pyasn1-0.5.0 pyasn1-modules-0.3.0 python-dateutil-2.8.2 requests-2.30.0 requests-oauthlib-1.3.1 rsa-4.9 six-1.16.0 urllib3-1.26.15 websocket-client-1.5.1
```

Once requirements are available, you'll typically use the role by launching the
`tests/kubelab.yml` playbook, like this:

```console
(ansible) user@lab ~ # ansible-playbook -i tests/inventory/kubelab tests/kubelab.yml
```

<u><strong>NOTE</strong></u>: date & time of the involved systems are important!
Having a clock skew between the machine you're executing Ansible playbooks and
the destination machines could cause certificate verification to fail.

<u><strong>NOTE</strong></u>: you can chose anytime to reset everything by
passing `k8s_reset` as `true`. This will reset your entire cluster, so use it
with caution:

```console
(ansible) user@lab ~ # ansible-playbook -i tests/inventory/kubelab tests/kubelab.yml -e k8s_reset=true
```

## Interact with the cluster after the installation

Once the playbook completes its execution the best way to interact with the
cluster is by using the `kubectl` command that can be installed as follows:

```console
user@lab ~ # curl -s -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

user@lab ~ # chmod +x kubectl

user@lab ~ # sudo mv kubectl /usr/local/bin
```

The Kubernetes role produces a local directory which contains the main
kubeconfig file, named admin.conf. The easiest way to use it is by exporting the
KUBECONFIG variable, like this:

```console
user@lab ~ # export KUBECONFIG=~/kubernetes/admin.conf
```

From now until the end of the session, every time you'll `kubectl` it will rely
on the credentials contained in that file:

```console
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

It is also possible to use different users to log into the cluster, check the
[Users](#Users) section for details.

## Configuration

### Inventory

A typical inventory depends on what you want to deploy, looking at the example
`kubelab`  you can declare inside the hosts file (see [tests/inventory/kubelab/hosts](https://github.com/mmul-it/ansible/blob/master/tests/inventory/kubelab/hosts))
all the nodes:

```ini
# Kubernetes hosts
[kubelab]
kubernetes-1 k8s_role=control-plane run_non_infra_pods=true
kubernetes-2 k8s_role=control-plane run_non_infra_pods=true
kubernetes-3 k8s_role=control-plane run_non_infra_pods=true
kubernetes-4 k8s_role=worker
```

You'll set which nodes will act as control plane and also whether or not those
will run non infrastructure pods (so to make the control plane also a worker).

Then you can define, inside group file (i.e.
[inventory/kubelab/group_vars/kubelab.yml](https://github.com/mmul-it/kubelab/blob/master/inventory/kubelab/group_vars/kubelab.yml)),
all the additional configurations, depending on what do you want to implement.

The name of the host group for the Kubernetes host is by default `kubelab` but
can be overridden by declaring the `k8s_host_group` variable.

### Kubernetes cluster

If you want to implement a multi-control-plane, high availability cluster
you'll need to specify these variables:

```yaml
k8s_cluster_name: kubelab

k8s_control_plane_node: kubernetes-1
k8s_control_plane_port: 6443
k8s_control_plane_cert_key: "91bded725a628a081d74888df8745172ed842fe30c7a3898b3c63ca98c7226fd"

k8s_multi_control_plane: true
k8s_balancer_VIP: 192.168.122.199
k8s_balancer_interface: eth0
k8s_balancer_port: 8443
k8s_balancer_password: "d6e284576158b1"

k8s_wait_timeout: 1200

k8s_control_plane_ports:
  - 2379-2380/tcp
  - 6443/tcp
  - 8443/tcp
  - 10250/tcp
  - 10257/tcp
  - 10259/tcp
```

This will bring up a cluster starting from node `kubernetes-1` enabling multi
control plane via `k8s_multi_control_plane` and setting the VIP address and the
interface.

**<u>Note</u>**: you'll want to change both `k8s_control_plane_cert_key` and
`k8s_balancer_password` for better security.

**<u>Note</u>**: it is possible to have a more atomic way to configure pods
network cidr, worker ports , nodeports ranges and firewall management, you can
check [the defaults file](https://github.com/mmul-it/ansible/blob/master/roles/kubernetes/defaults/main.yml#L50-L67).

### Network Add-on

The Kubernetes role supports Flannel and Calico network add-ons. The
configuration depends on which add-on you want to implement.

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

Once the installation completes the easiest way to access the dashboard is by
using the `kubectl proxy` and then access the related url
[http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/).

A login prompt will be presented, and you can login by passing a token. By
default the Kubernetes role creates a user named `dashboard-user` (you can
override it).

To retrieve the token you'll need use `kubectl`, like this:

```console
user@lab ~ # kubectl -n kubernetes-dashboard create token dashboard-user
<YOUR TOKEN>
```

Copy and paste the output of the above command inside the prompt and you'll
complete the login.

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

```console
user@lab ~ # ls -1 kubernetes/users/
pod-viewer.crt
pod-viewer.csr
pod-viewer.key
users.conf
users-rolebindings.yaml
users-roles.yaml
```

The `users.conf` file can then be used to access the cluster with this user,
like this:

```console
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

The Kubernetes role actually supports the implementation of the Ceph CSI
StorageClass. It can be defined as follows:

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

To enable [MetalLB](https://metallb.universe.tf/) a load-balancer implementation
for baremetal Kubernetes clusters, using standard routing protocols, it is
sufficient to declare:

```yaml
k8s_metallb_enable: true
k8s_metallb_pools:
  - name: 'first-pool'
    addresses: '192.168.122.100-192.168.122.130'
```

Then it will be possible to use this LoadBalancer to create IPs inside the
declared pool range (check next `ingress-nginx` example to understand how).

### Ingress NGINX

To enable [Ingress NGINX](https://github.com/kubernetes/ingress-nginx), an
Ingress controller for Kubernetes using NGINX as a reverse proxy and load
balancer, it is sufficient to declare:

```yaml
k8s_ingress_nginx_enable: true
```

This will install the Ingress NGINX controller that can be used for different
purposes.

#### Ingress NGINX on control-planes

For example it is possible to use Ingress NGINX by exposing the `80` and `443`
ports on the balanced IP managed by haproxy, by declaring this:

```yaml
k8s_ingress_nginx_enable: true
k8s_ingress_nginx_haproxy_conf: true
k8s_ingress_nginx_services:
  - name: ingress-nginx-externalip
    spec:
      externalIPs:
      - 192.168.122.199
      ports:
      - name: port-1
        port: 80
        protocol: TCP
      - name: port-2
        port: 443
        protocol: TCP
      selector:
        app.kubernetes.io/component: controller
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/name: ingress-nginx
```

This will expose both ports on the balanced IP (in this case `192.168.122.199`
and will make the service responding there.

To test it just try this:

```console
$ kubectl create deployment demo --image=httpd --port=80
deployment.apps/demo created

$ kubectl expose deployment demo
service/demo exposed

$ kubectl create ingress demo --class=nginx --rule="demo.192.168.122.199.nip.io/*=demo:80"
ingress.networking.k8s.io/demo created

$ curl http://demo.192.168.122.199.nip.io
<html><body><h1>It works!</h1></body></html>
```

#### Ingress NGINX with MetalLB

Another way is to use it in combination with MetalLB, by declaring a
`LoadBalancer` service, as follows:

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

This will install everything related to the controller, and assign the
`loadBalancerIP` that is part of the range supplied by MetalLB, by exposing both
`80` and `443` ports.

### Cert Manager

To enable [Cert Manager](https://cert-manager.io/), a controller to automate
certificate management in Kubernetes, it is sufficient to declare:

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

This will install everything related to the controller and create a cluster
issuer that will use `letsencrypt` with `http01` challenge resolution, via NGINX
ingress class.

Once everything is installed and you want to expose an application, you can test
everything by using something like this yaml:

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

If you look specifically on the last resource, the `Ingress` named
`nginx-ingress` you will see two important sections:

- Under `metadata` -> `annotations` the annotation
  `cert-manager.io/cluster-issuer: letsencrypt`

- Under `spec:` -> `tls` the host declaration.

With this in place, after some time, you'll have your cert served for the
exposed service.

## License

MIT

## Author Information

Raoul Scarazzini ([rascasoft](https://github.com/rascasoft))
