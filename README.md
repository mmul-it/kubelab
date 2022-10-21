# MMUL Ansible roles

This collection of roles can be used to implement various cloud native
solutions:

* [Kubernetes](roles/kubernetes): a Kubernetes cluster;
* [Terraform](roles/terraform): based on the passed inventory, it generates
  all the Terraform files needed to implement a cloud (today just Azure)
  datacenter;
* [Azure](roles/azure): automate tasks on Azure;
* [Graylog](roles/graylog-server): a clustered Graylog/MongoDB/Elasticsearch
  solution;
* [MaxScale](roles/maxscale): a MariaDB MaxScale instance (suitable also for
  Azure MaxScale);
* [Redis](roles/redis): a Redis cluster implementation;

A specific README will be (soon, if not present) available for each role.

## Using the roles

For each of the role there is an Ansible plugin that will invoke everything that is needed to accomplish its scope.

### Preparing the environment

The best way to run everything is by using Python virtual environments and Ansible Collections.

First of all you need to clone this repository:

```
user@lab ~ # git clone https://github.com/mmul-it/ansible
Cloning into 'ansible'...
remote: Enumerating objects: 2572, done.
remote: Counting objects: 100% (449/449), done.
remote: Compressing objects: 100% (199/199), done.
remote: Total 2572 (delta 269), reused 363 (delta 222), pack-reused 2123
Receiving objects: 100% (2572/2572), 9.10 MiB | 8.77 MiB/s, done.
Resolving deltas: 100% (1345/1345), done.
```

Then you will create your Python virtual environment:

```
user@lab ~ # python3 -m virtualenv ansible-env
Running virtualenv with interpreter /usr/bin/python2
New python executable in /root/ansible-env/bin/python2
Also creating executable in /root/ansible-env/bin/python
Installing setuptools, pkg_resources, pip, wheel...done.

# source ansible-env/bin/activate
(ansible-env) user@lab ~ #

(ansible-env) user@lab ~ # python3 -m pip install --upgrade pip
```

And then you'll be ready to install the Python requirements:

```
(ansible-env) user@lab ~ # pip3 install -r ansible/requirements.txt 
Requirement already satisfied: ansible in /usr/lib/python3.6/site-packages (from -r ansible/requirements.txt (line 1)) (2.10.7)
Requirement already satisfied: ansible-vault in /usr/lib/python3.6/site-packages (from -r ansible/requirements.txt (line 2)) (1.2.0)
Requirement already satisfied: azure-cli in /usr/lib/python3.6/site-packages (from -r ansible/requirements.txt (line 3)) (2.25.0)
...
...
Installing collected packages: rsa, cachetools, google-auth, kubernetes
Successfully installed cachetools-4.2.4 google-auth-2.13.0 kubernetes-24.2.0 rsa-4.9
```

And finally the Ansible collections:

```
(ansible-env) user@lab ~ # ansible-galaxy install -r ansible/collections/requirements.yml
Starting galaxy collection install process
Process install dependency map
...
...
kubernetes.core (2.3.2) was installed successfully
```

Now you should be ready to use `ansible-playbook` to execute the desired playbooks.

### Launching the playbooks

Supposing that you want to install a Kubernetes environment, you'll need to pass an inventory (we will rely on `lab`) like this:

```
$ ansible-playbook \
-i $HOME/ansible/inventory/lab \
$HOME/ansible/kubernetes.yml
```

## Authors

This project was created and is maintained by [Raoul Scarazzini](https://github.com/rascasoft) and has contributions from other users. Thanks to everyone who will contribute in the future, you're more than welcome!

## License

MIT License
