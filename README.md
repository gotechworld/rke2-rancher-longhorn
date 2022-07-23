

# RKE2, Longhorn, and Rancher Install


## Prerequisites

The prerequisites are fairly simple. We need 3 linux servers with access 
to the internet. They can be bare metal, or in the cloud provider of your 
choice. I prefer [Digital Ocean](https://digitalocean.com). We need an 
`ssh` client to connect to the servers. And finally DNS to make things 
simple. Ideally we need a URL for the Rancher interface. For the purpose 
of the this guide let's use `rancher.dockr.life`. We will need to point 
that name to the first server of the cluster. While we are at it, a 
wildcard DNS for your domain will help as well.

## Linux Servers

For the sake of this guide we are going to use 
[Ubuntu](https://ubuntu.com). Our goal is a simple deployment. The 
recommended size of each node is 4 Cores and 8GB of memory with at least 
60GB of storage. One of the nice things about 
[Longhorn](https://longhorn.io) is that we do not need to attach 
additional storage. Here is an example list of servers. Please keep in 
mind that your server names can be anything. Just keep in mind which ones 
are the "server" and "agents".

| name | ip | memory | core | disk | os |
|---| --- | --- | --- | --- | --- |
|rancher1| 142.93.189.52  | 8192 | 4 | 160 | Ubuntu 21.10 x64 |
|rancher2| 68.183.150.214 | 8192 | 4 | 160 | Ubuntu 21.10 x64 |
|rancher3| 167.71.188.101 | 8192 | 4 | 160 | Ubuntu 21.10 x64 |


For Kubernetes we will need to "set" one of the nodes as the control 
plane. Rancher1 looks like a winner for this. First we need to `ssh` into 
all three nodes and make sure we have all the updates and add a few 
things. For the record I am not a fan of software firewalls. Please feel 
free to reach to me to discuss. :D

**Ubuntu**:

```bash
# Ubuntu instructions 
# stop the software firewall
systemctl stop ufw
systemctl disable ufw

# get updates, install nfs, and apply
apt update
apt install nfs-common -y  
apt upgrade -y

# clean up
apt autoremove -y
```

**Rocky / Centos / RHEL**:

```bash
# Rocky instructions 
# stop the software firewall
systemctl stop firewalld
systemctl disable firewalld

# get updates, install nfs, and apply
yum install -y nfs-utils cryptsetup iscsi-initiator-utils

# enable iscsi for Longhorn
systemctl start iscsid.service
systemctl enable iscsid.service

# update all the things
yum update -y

# clean up
yum clean all
```

Cool, lets move on to the RKE2.

## RKE2 Install

### RKE2 Server Install

Now that we have all the nodes up to date, let's focus on `rancher1`. 
While this might seem controversial, `curl | bash` does work nicely. The 
install script will use the tarball install for **Ubuntu** and the RPM 
install for **Rocky/Centos**. Please be patient, the start command can 
take a minute. Here are the [rke2 
docs](https://docs.rke2.io/install/methods/) and [install 
options](https://docs.rke2.io/install/install_options/install_options/) 
for reference.

```bash
# On rancher1
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=server sh - 

# start and enable for restarts - 
systemctl enable rke2-server.service 
systemctl start rke2-server.service
```

Here is what the **Ubuntu** version should look like:

Let's validate everything worked as expected. Run a `systemctl status 
rke2-server` and make sure it is `active`.

Perfect! Now we can start talking Kubernetes. We need to symlink the 
`kubectl` cli on `rancher1` that gets installed from RKE2.

```bash
# simlink all the things - kubectl
ln -s $(find /var/lib/rancher/rke2/data/ -name kubectl) 
/usr/local/bin/kubectl

# add kubectl conf
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml 

# check node status
kubectl  get node
```

Hopefully everything looks good! Here is an example.

For those that are not TOO familiar with k8s, the config file is what 
`kubectl` uses to authenticate to the api service. If you want to use a 
workstation, jump box, or any other machine you will want to copy 
`/etc/rancher/rke2/rke2.yaml`. You will want to modify the file to change 
the ip address. We will need one more file from `rancher1`, aka the 
server, the agent join token. Copy 
`/var/lib/rancher/rke2/server/node-token`, we will need it for the agent 
install.

Side note on Tokens. RKE2 uses the TOKEN as a way to authenticate the 
agent to the server service. This is a much better system than "trust on 
first use". The goal of the token process is to setup a control plane 
Mutual TLS (mtls) certificate termination.

### RKE2 Agent Install

The agent install is VERY similar to the server install. Except that we 
need an agent config file before starting. We will start with `rancher2`. 
We need to install the agent and setup the configuration file.

```bash
# we add INSTALL_RKE2_TYPE=agent
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=agent sh -  

# create config file
mkdir -p /etc/rancher/rke2/ 

# change the ip to reflect your rancher1 ip
echo "server: https://$RANCHER1_IP:9345" > /etc/rancher/rke2/config.yaml

# change the Token to the one from rancher1 
/var/lib/rancher/rke2/server/node-token 
echo "token: $TOKEN" >> /etc/rancher/rke2/config.yaml

# enable and start
systemctl enable rke2-agent.service
systemctl start rke2-agent.service
```

What should this look like:

Rinse and repeat. Run the same install commands on `rancher3`. Next we can 
validate all the nodes are playing nice by running `kubectl get node -o 
wide` on `rancher1`. 

Huzzah! RKE2 is fully installed. From here on out we will only need to 
talk to the kubernetes api. Meaning we will only need to remain ssh'ed 
into `rancher1`.

## Rancher

For more information about the Rancher versions, please refer to the  
[Support 
Matrix](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-6-3/). 
We are going to use the latest version. For additional reading take a look 
at the [Rancher docs](https://rancher.com/docs/rancher/v2.6/en/).

### Rancher Install

For Rancher we will need [Helm](https://helm.sh/). We are going to live on 
the edge! Here are the [install 
docs](https://rancher.com/docs/rancher/v2.6/en/installation/install-rancher-on-k8s/) 
for reference.

```bash
# on the server rancher1
# add helm
curl -#L https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# add needed helm charts
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo add jetstack https://charts.jetstack.io
```

Quick note about Rancher. Rancher needs jetstack/cert-manager to create 
the self signed TLS certificates. We need to install it with the Custom 
Resource Definition (CRD). Please pay attention to the `helm` install for 
Rancher. The URL will need to be changed to fit your FQDN. Also notice I 
am setting the `bootstrapPassword` and replicas. This allows us to skip a 
step later. 

```bash
# still on  rancher1
# add the cert-manager CRD
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.8.2/cert-manager.crds.yaml

# helm install jetstack
helm upgrade -i cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace

# helm install rancher
helm upgrade -i rancher rancher-latest/rancher --create-namespace --namespace cattle-system --set hostname=rancher.petrugiurca.net \
--set bootstrapPassword=bootStrapAllTheThings \
--set ingress.tls.source=letsEncrypt \
--set letsEncrypt.email=petru.giurca@pm.me \
--set letsEncrypt.ingress.class=nginx \
--set replicas=3
```


For information on how to configure cert-manager to automatically 
provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:

https://cert-manager.io/docs/usage/ingress/

We can also run a `kubectl get pod -A` to see if everything it running. 
Keep in mind it may take a minute or so for all the pods to come up.

### Rancher Design

Let's take a second and talk about Ranchers Multi-cluster design. Bottom 
line, Rancher can operate in a Spoke and Hub model. Meaning one k8s 
cluster for Rancher and then "downstream" clusters for all the workloads. 
Personally I prefer the decoupled model where there is only one cluster 
per Rancher install. This allows for continued manageability during 
networks outages. For the purpose of the is guide we are concentrate on 
the single cluster deployment. There is good 
[documentation](https://rancher.com/docs/rancher/v2.6/en/cluster-provisioning/registered-clusters/) 
on "importing" downstream clusters.

## Longhorn

### Lognhorn Install

There are two methods for installing. Rancher has Chart built in.

Now for the good news, [Longhorn 
docs](https://longhorn.io/docs/1.2.4/deploy/install/) show two easy 
install methods. Helm and `kubectl`. Let's stick with Helm for this guide.

```bash
# get charts
helm repo add longhorn https://charts.longhorn.io

# update
helm repo update

# install
helm upgrade -i longhorn longhorn/longhorn --namespace longhorn-system --create-namespace
```

One of the other benefits of this integration is that rke2 also knows it 
is installed. Run `kubectl get sc` to show the storage classes.

```text
root@rancher1:~# kubectl  get sc
NAME                 PROVISIONER          RECLAIMPOLICY   
VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
longhorn (default)   driver.longhorn.io   Delete          Immediate           
true                   3m58s
```

Now we have a default storage class for the cluster. This allows for the 
automatic creation of Physical Volumes (PVs) based on a Physical Volume 
Claim (PVC). The best part is that "it just works" using the existing, 
unused storage, on the three nodes. Take a look around in the gui. Notice 
the Volumes on the Nodes. For fun, here is a demo flask app that uses a 
PVC for Redis. `kubectl apply -f 
https://raw.githubusercontent.com/clemenko/k8s_yaml/master/flask_simple_nginx.yml`



