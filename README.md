# How to install RKE2 on Ubuntu 22.04 (one Master, one Worker node)

## Introduction:
RKE2, also known as RKE Government, is Rancher's next-generation Kubernetes distribution.

Channel URL for fetching RKE2 download URL. Defaults to https://update.rke2.io/v1-release/channels
<br>


<br></br>


## Prerequisites
Make sure your environment fulfills the [requirements](https://docs.rke2.io/install/requirements).
- Two rke2 nodes cannot have the same node name. (By default, the node name is taken from the machine's hostname.)
- If your node has "NetworkManager" installed and enabled, [ensure that it is configured to ignore CNI-managed interfaces](https://docs.rke2.io/known_issues#networkmanager).
- The RKE2 server needs ports `6443` and `9345` to be accessible by other nodes in the cluster.
- The RKE2 installation process must be run as the `root` user or through `sudo`.
- [Inbound Network Rules](https://docs.rke2.io/install/requirements#inbound-network-rules)


```bash
Note:
As such, if installing RKE2 on a NetworkManager enabled system, it is highly recommended to configure NetworkManager to ignore calico/flannel related network interfaces.


# to check state of NetworkManager
systemctl status NetworkManager.service

# if its enable, please follow the below structure
cat<<-EOF > /etc/NetworkManager/conf.d/rke2-canal.conf
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:flannel*
EOF

systemctl reload NetworkManager.service

# important note: 
If performing this configuration change on a system that already has RKE2 installed, a reboot of the node is necessary to effectively apply the changes.
```

<br></br>

```bash
# --> [ on all nodes ] <--
sudo su -
apt update -y
apt install net-tools vim wget curl bash-completion -y
snap install yq
# source it from ~/.bashrc or ~/.bash_profile
echo "source /etc/profile.d/bash_completion.sh" >> ~/.bashrc

# check time, and kernel version, all nodes must be same
timedatectl
ls -l /etc/localtime
df -h
cat /etc/os-release

# have below comamnd and document it somewhere later weill be needed.
cat /etc/hosts
cat /etc/resolv.conf
uname -a
```

```bash
# --> [ master node ] <--
hostnamectl set-hostname master-2-001 && bash
timedatectl set-timezone Europe/London && timedatectl
systemctl status NetworkManager.service

ip a
# check conectivity between nodes
ping <WORKER_IP>


# --> [ worker node ] <--
hostnamectl set-hostname worker-2-001 && bash
timedatectl set-timezone Europe/London && timedatectl
systemctl status NetworkManager.service

ip a
# check conectivity between nodes
ping <MASTER_IP>

mkdir apps-to-deploy && cd apps-to-deploy
```




<br></br><br></br><br></br>
# Server / Master Node Installation:
RKE2 provides an installation script that is a convenient way to install it as a service on systemd based systems.

To pick a version, follow this [doc](https://github.com/rancher/rke2/releases).
```bash
# This will install the rke2-server service and the rke2 binary onto your machine.
--> This release updates Kubernetes to v1.27.12. <--
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="server" INSTALL_RKE2_CHANNEL=stable INSTALL_RKE2_VERSION=v1.27.12+rke2r1 sh -

echo $?
# To get more info of systemd service
systemctl cat rke2-server.service

# Enable the rke2-server service
systemctl enable rke2-server.service

# Start the service
systemctl start rke2-server.service && systemctl status rke2-server.service

# Follow the logs, if you like
journalctl -u rke2-server -f
echo $?

# check the ports
netstat -ntlp

# copy the token, will need it to join worker nodes
cat /var/lib/rancher/rke2/server/token

# to have access kubectl command:
echo "export KUBECONFIG=/etc/rancher/rke2/rke2.yaml" >> ~/.bashrc
echo "export PATH=$PATH:/var/lib/rancher/rke2/bin" >> ~/.bashrc

# https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#enable-shell-autocompletion
echo 'source <(kubectl completion bash)' >>~/.bashrc
kubectl get node -owide
kubectl get pods --all-namespaces
```

## Note:
> A kubeconfig file will be written to `/etc/rancher/rke2/rke2.yaml`.

> A token that can be used to register other server or agent nodes will be created at `/var/lib/rancher/rke2/server/node-token`

> Two cleanup scripts, `rke2-killall.sh` and rke2-uninstall.sh, will be installed to the path at:
`/usr/local/bin` for regular file systems
`/opt/rke2/bin` for read-only and brtfs file systems


<br></br>
# Worker / Agent Node Installation:

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" INSTALL_RKE2_CHANNEL=stable INSTALL_RKE2_VERSION=v1.27.12+rke2r1 sh -
echo $?

# To get more info of systemd service
systemctl cat rke2-agent.service

# Enable the rke2-agent service
systemctl enable rke2-agent.service

# NOTE
The rke2 server process listens on port "9345" for new nodes to register. The Kubernetes API is still served on port "6443", as normal.

# Configure the rke2-agent service
mkdir -p /etc/rancher/rke2/
vim /etc/rancher/rke2/config.yaml
server: https://<server>:9345
token: <token from server node>


# Start the service
systemctl start rke2-agent.service && systemctl status rke2-agent.service

# To see the state of worker node, run on "master" node:
kubectl get node -w

# Follow the logs, if you like
journalctl -u rke2-agent -f
```


<br></br>
## Cluster Access and to confirm cluster is functional
The kubeconfig file stored at `/etc/rancher/rke2/rke2.yaml` is used to configure access to the Kubernetes cluster. 

```bash
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export PATH=$PATH:/var/lib/rancher/rke2/bin
kubectl get pods --all-namespaces
```

### Accessing the Cluster from Outside with kubectl
> Copy `/etc/rancher/rke2/rke2.yaml` on your machine located outside the cluster as `~/.kube/config`. Then replace `127.0.0.1` with the IP or hostname of your RKE2 server. kubectl can now manage your RKE2 cluster.

<br></br><br></br><br></br><br></br>

## Deploy Wordpress and Mysql to test functionality of our cluster

```bash
# --> [ master node ] <--
# Step 1: Create MySQL StatefulSet
We will begin by creating a MySQL StatefulSet. Save the following content to a file named mysql-statefulset.yaml:

# Step 2: Create WordPress Deployment and Service
Now, create a WordPress Deployment YAML file. Save the following content to a file named wordpress-deployment.yaml: Here you have deployment and service files

# Step 3: Create a Secret for MySQL Password
Create a Secret to securely store the MySQL root password. Save the following content to a file named mysql-secret.yaml

# Step 4: Create Storage class, Persistent Volume and Persistent Volume Claim for MySQL and WordPress
To ensure data persistence for both MySQL and WordPress, create Persistent Volumes and Persistent Volume Claims.


# Step 5: Apply the Configurations
kubectl create ns dev
kubectl -n dev apply -f storageclass.yaml
kubectl -n dev apply -f mysql-secret.yaml
kubectl -n dev get secrets mysql-secret -oyaml | yq '.data|map_values(@base64d)'
kubectl -n dev apply -f mysql-pv-pvc.yaml
kubectl -n dev apply -f mysql-statefulset.yaml
kubectl -n dev apply -f wordpress-pv-pvc.yaml
kubectl -n dev apply -f wordpress-deployment.yaml
kubectl -n dev get pv,pvc,pod

# Access Your WordPress Site
kubectl get svc wordpress-service

# To get access Wordpress in your browser enter:
http://<MASTER_NODE_IP>:30007
```

```bash
# to make our applicaition available all the time, we need to define Disruption Budget for both MySQL and WordPress
kubectl -n dev apply -f pdb-mysql.yaml
kubectl -n dev apply -f pdb-wordpress.yaml

kubectl -n dev get pdb
```

<br></br>
:warning:
> Before starting the upgrade we need to check all changes, and the deprecated API of our applications was deployed on the cluster on.



<br></br><br></br>

# [Manual Upgrades](https://docs.rke2.io/upgrade/manual_upgrade)
Upgrade the "server" nodes first, one at a time. Once all servers have been upgraded, you may then upgrade "agent" nodes. we are going to upgrade from `v1.27.12` to `v1.28.8`.

> Please before stating any upgrade, read [Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/)

### Before starting, needed to mention a couple of solutions to achieve zero downtime(since we have a stateful set of apps):
- Add extra worker node and install the new version required, then drain workload into a new node.
- Allow scheduling of pods on Kubernetes master. using taint
- Using PDB for the stateful applications.
- Copy the binary file, to the nodes and reload the process OR use the below structure.


### On master node(s):
```bash
rke2 --version

# [in other terminals run] to track the status of applications and clsuter
journalctl -u rke2-server -f
kubectl get nodes -w
kubectl -n dev get pod -owide -w


curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="server" INSTALL_RKE2_CHANNEL=stable INSTALL_RKE2_VERSION=v1.28.8+rke2r1 sh -

echo $?

# restart the rke2 process
systemctl status rke2-server.service
systemctl restart rke2-server.service && systemctl status rke2-server.service
rke2 --version
```



### On worker node(s):
```bash
rke2 --version

# [in other terminals run] to track the status of applications and clsuter
journalctl -u rke2-agent -f
kubectl get nodes -w
kubectl -n dev get pod -owide -w


curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" INSTALL_RKE2_CHANNEL=stable INSTALL_RKE2_VERSION=v1.28.8+rke2r1 sh -
echo $?

# restart the rke2 process
systemctl status rke2-agent.service
systemctl restart rke2-agent.service && systemctl status rke2-agent.service
rke2 --version

```



<br></br><br></br>
## [Configuration Options](https://docs.rke2.io/install/configuration)
The primary way to configure RKE2 is through its config file.

By default, RKE2 will launch with the values present in the YAML file located at `/etc/rancher/rke2/config.yaml`.
If the configuration is changed after starting RKE2, the service must be restarted to apply the new configuration.

```bash
# to download the install script and make it executable.
curl -sfL https://get.rke2.io --output install.sh
chmod +x install.sh

INSTALL_RKE2_CHANNEL=latest ./install.sh
```


<br></br>
## To Uninstall
Uninstalling RKE2 deletes the cluster data and all of the scripts.

### Tarball Method
To uninstall RKE2 installed via the Tarball method from your system, simply run the command below. This will terminate the process, remove the RKE2 binary, and clean up files used by RKE2.
```bash
/usr/local/bin/rke2-uninstall.sh
```


<br></br>
# To Do / Consideration

### NOTE:
If you are adding additional server nodes, you must have an odd number in total. An odd number is needed to maintain quorum.


<br></br>
## Referenaces:
https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/

Official doc: https://docs.rke2.io/

[Pod disruption budgets](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#pod-disruption-budgets)

[K8s Volumes](https://kubernetes.io/docs/concepts/storage/volumes/#local)