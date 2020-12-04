# CKAD Cluster Setup and Training Resources

This is a fork of the [kubeadm-ansible](https://github.com/kairen/kubeadm-ansible) repo that spins up a Kubernetes cluster using Ansible with kubeadm. I used it as preparation for the [Certified Kubernetes Application Developer (CKAD) Program](https://www.cncf.io/certification/ckad/) to have an easy-to-(re)create environment to play around with.

The cluster setup playbook has been tested successfully with the following configuration:

* [Kubernetes 1.19](https://kubernetes.io/docs/setup/release/notes/#v1-19-0) 
* [Docker (docker-ce) 18.06](https://docs.docker.com/engine/release-notes/)
* [Calico Networking](https://www.projectcalico.org/) intead of flannel 
* Two [Ubuntu 18.04 LTS](https://ubuntu.com/download/server) small sized machines (2 vCPU, 2 GiB RAM), one acting as master and one as worker node

 I've used servers managed by [Linux Academy Cloud Playground](https://linuxacademy.com/) as they also provide a dedicated CKAD Training, but could use any cloud provider or on premise infrastructure. Remember you need to perform some intial ssh setup before running the playbook, see *System requirements* below

# Useful CKAD Resources

## Snippets to speed up kubectl interaction

```
# useful aliases
alias kc=kubectl
alias kn='kubectl config set-context --current --namespace '
alias pods="kubectl get pods"
alias ke="kubectl explain --recursive"

export $dry="--dry-run=client -o yaml" # for kubectl run quick yaml export
export $force="--force --grace-period=0" # to speed up kubectl delete xy

# auto complete
source <(kubectl completion bash)
complete -F __start_kubectl kc # enable also for kc alias

# tune vim tabstop, softtabstop, shiftwdith and tabs=>spaces
echo "set ts=2 sts=2 sw=2 et" > ~./.vimrc && . ~/.vimrc
# extended version below
```
```
cat <<EOF >>~/.vimrc 
autocmd FileType yml,yaml setlocal ts=2 sts=2 sw=2 expandtab indentkeys-=0# indentkeys-=<:>
EOF
```

## Vim yaml config explained

[tip](https://stackoverflow.com/questions/26962999/wrong-indentation-when-editing-yaml-in-vim)
For YAML file it instructs Vim to use 2 spaces for indentation, Use spaces instead of tabs and
Skip re-indenting lines after inserting a comment character (#) at the beginning of a line, or a colon.
vim mark lines: `Esc+V` (then arrow keys), Copy marked lines: `y`, cut: `d`, Paste: `p` or `P`
 
## Snippets for daily work

```
kc explain po; kc explain po.spec; kc explain pod.spec.volumes

# export is apparently gone with 1.19 :-(
kubectl get pod coredns-42 -n kube-system -o yaml --export >backup.yaml


# create skeleton yaml for deployment
kc create deployment q1 --image=nginx --dry-run=client -o yaml >q1-tmpl.yaml

# (...)
```

## Exam Prep External Link Collection

* [Practice Exam for Certified Kubernetes Application Developer (CKAD) Certification](https://matthewpalmer.net/kubernetes-app-developer/articles/ckad-practice-exam.html)
* [List of resources and notes for passing the Certified Kubernetes Application Developer (CKAD) exam.](https://github.com/twajr/ckad-prep-notes)
* [CKAD Tips](https://pnguyen.io/posts/ckad-tips/) and [CKAD Exercises](https://github.com/dgkanatsios/CKAD-exercises)
* [The CKAD browser terminal](https://codeburst.io/the-ckad-browser-terminal-10fab2e8122e) and [simulator](https://killer.sh/)
* [Candidate Handbook (official)](https://training.linuxfoundation.org/wp-content/uploads/2019/04/CKA-CKAD-Candidate-Handbook-v1.18-March-2019.pdf)
* [Important tips (official)](https://training.linuxfoundation.org/wp-content/uploads/2019/05/Important-Tips-CKA-CKAD-4.30.19.pdf)
* [Official Exam Resources](https://www.cncf.io/certification/ckad/) especially [curriculum](https://github.com/cncf/curriculum), [exam tips](https://training.linuxfoundation.org/wp-content/uploads/2020/01/Important-Tips-CKA-CKAD-01.28.2020.pdf) and [faq](https://training.linuxfoundation.org/wp-content/uploads/2020/01/CKA-CKAD-FAQ-01.28.2020.pdf)
* [official kubectl cheatsheet](https://kubernetes.io/de/docs/reference/kubectl/cheatsheet/)

# Automated Kubernetes Cluster Setup

## System requirements

* Deployment environment must have Ansible `2.4.0+` (`pip install --user ansible`)
* Master and nodes must have passwordless SSH access. For ssh login you can easily create a keypair and add the public key to remote `~/.ssh/authorized_keys`.
 ```
    # both private and public key are placed in .secret and git-ignored 
    ssh-keygen -t rsa -b 4096 -f .secret/id_rsa  -N ""
    ssh-copy-id -i .secret/id_rsa.pub cloud_user@till1.server.com # repeat or each host
```
* For easy access you can setup an custom Host entry in your `~/.ssh/config` file
```
Host *.server.com 
  User cloud_user
  IdentityFile ~/path/to/ckad-cluster/.secret/id_rsa
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
```
  
* Since ansible needs to execute some commands with elevated privileges, you may also have to use Ansible's `--ask-become-pass` option or store it in `hosts.ini` (not recommended) 
  
## Customization

Add the system information gathered above into a file called `hosts.ini`, you can use `hosts.ini.tmpl` as a template and just adapt hostnames and ssh config

Also adapt `group_vars/all.yml` to your specified configuration.
For example, pick a different version of Kubenernetes or choose `flannel` instead of `calico`
To update docker version, check [available versions](https://download.docker.com/linux/static/stable) and update [roles/docker/defaults/main.yml](roles/docker/defaults/main.yml) accordingly.

**Note:** Depending on your setup, you may need to modify `cni_opts` to an available network interface. By default, `kubeadm-ansible` uses `eth1`. Your default interface may be `eth0`.

After going through the setup, run the `site.yaml` playbook:

```sh
$ ansible-playbook site.yaml
...
kubernetes/master : Init Kubernetes cluster -------------------------------------------------------------------------------------------------------------------------------- 51.30s
kubernetes/node : Recreate kube-dns ---------------------------------------------------------------------------------------------------------------------------------------- 21.63s
commons/pre-install : Install kubernetes packages (Debian/Ubuntu) ---------------------------------------------------------------------------------------------------------- 19.56s
commons/pre-install : Install kubernetes packages (Debian/Ubuntu) ---------------------------------------------------------------------------------------------------------- 18.10s
docker : Install docker engine (Debian/Ubuntu) ----------------------------------------------------------------------------------------------------------------------------- 15.32s
docker : Install apt-transport-https --------------------------------------------------------------------------------------------------------------------------------------- 13.02s
docker : Add Docker APT repository ------------------------------------------------------------------------------------------------------------------------------------------ 8.62s
commons/pre-install : Add Kubernetes APT repository ------------------------------------------------------------------------------------------------------------------------- 7.59s
commons/pre-install : Add Kubernetes APT repository ------------------------------------------------------------------------------------------------------------------------- 7.45s
kubernetes/node : Join to Kubernetes cluster -------------------------------------------------------------------------------------------------------------------------------- 6.74s
Gathering Facts ------------------------------------------------------------------------------------------------------------------------------------------------------------- 4.60s
Gathering Facts ------------------------------------------------------------------------------------------------------------------------------------------------------------- 4.29s
commons/pre-install : Disable swappiness and pass bridged IPv4 traffic to iptable's chains ---------------------------------------------------------------------------------- 3.30s
commons/pre-install : Disable swappiness and pass bridged IPv4 traffic to iptable's chains ---------------------------------------------------------------------------------- 3.27s
docker : Copy Docker engine service file ------------------------------------------------------------------------------------------------------------------------------------ 3.12s
docker : Copy Docker environment config file -------------------------------------------------------------------------------------------------------------------------------- 2.64s
cni : Copy calico YAML files ------------------------------------------------------------------------------------------------------------------------------------------------ 2.50s
commons/pre-install : Copy kubeadm conf to drop-in directory ---------------------------------------------------------------------------------------------------------------- 2.49s
Gathering Facts ------------------------------------------------------------------------------------------------------------------------------------------------------------- 2.46s
commons/pre-install : Copy kubeadm conf to drop-in directory ---------------------------------------------------------------------------------------------------------------- 2.42s
```

The playbook will download `/etc/kubernetes/admin.conf` file to `.secret/admin.conf` from master and populate it to 
`~/.kube/conf` on each node (which is the default location so you don't need to specify `KUBECONFIG`) environment variable

Verify cluster is fully running using kubectl:

```sh
$ kubectl get node
NAME                         STATUS   ROLES    AGE   VERSION
till1.server.com             Ready    master   23m   v1.17.2
till2.server.com             Ready    <none>   17m   v1.17.2
```
```
$ kubectl get po --all-namespaces
NAME                                    READY     STATUS    RESTARTS   AGE
etcd-master1                            1/1       Running   0          23m
...
```
```
$ kubectl cluster-info
Kubernetes master is running at https://172.xx.xx.xx:6443
KubeDNS is running at https://172.xx.xx.xx:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```
```
$ kubectl run nginx --image=nginx
$ kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          13s
```

## Resetting the environment

Finally, reset all kubeadm installed state using `reset-site.yaml` playbook:

```sh
$ ansible-playbook reset-site.yaml
```

# Additional features
These are features that you could want to install to make your life easier.

Enable/disable these features in `group_vars/all.yml` (all disabled by default):
```
# Additional feature to install
additional_features:
  healthcheck: false
```

## Healthcheck
This will install k8s-healthcheck (https://github.com/emrekenci/k8s-healthcheck), a small application to report cluster status.
