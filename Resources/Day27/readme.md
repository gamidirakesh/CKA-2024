# Day 27/40 - Setup a Multi Node Kubernetes Cluster Using Kubeadm

## Check out the video below for Day27 👇

[![Day 27/40 - Setup a Multi Node Kubernetes Cluster Using Kubeadm ](https://img.youtube.com/vi/WcdMC3Lj4tU/sddefault.jpg)](https://youtu.be/WcdMC3Lj4tU)


#### What is kubeadm
`kubeadm` is a tool to bootstrap the Kubernetes cluster, which installs all the control plane components and gets the cluster ready for you.

Along with control plane components(ApiServer, ETCD, Controller Manager, Scheduler) , it helps with the installation of various CLI tools such as Kubeadm, Kubelet and kubectl

### Ways to install Kubernetes

![image](https://github.com/user-attachments/assets/5391de53-36bc-4574-81cf-4b1fefbda9e3)

>Note: In this demo, we will be installing Kubernetes on VMs on cloud(Self-Managed)

### Steps to set up the Kubernetes cluster

![image](https://github.com/user-attachments/assets/e0943ad5-2d13-4128-8147-1ef644c62955)


>Note: If using a Mac Silicon chip, Multipass is the recommended choice as Virtualbox has some compatibility issues or you can spin up virtual machines on the cloud and use that


**If you are using AWS EC2 servers, you need to allow specific traffic on specific ports as below**

1) Provision 3 VMs, 1 Master, and 2 Worker nodes.
2) Create 2 security groups, attach 1 to the master and the other to two worker nodes using the below details.
https://kubernetes.io/docs/reference/networking/ports-and-protocols/

![image](https://github.com/user-attachments/assets/58d66bcb-ed00-453f-99b2-8df9f4393cac)


4) If you are using AWS EC2, you need to disable **source destination check** for the VMs using the [doc](https://docs.aws.amazon.com/vpc/latest/userguide/work-with-nat-instances.html#EIP_Disable_SrcDestCheck)

### Run the below steps on the Master VM
1) SSH into the Master EC2 server

2)  Disable Swap using the below commands
```bash
swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
3) Forwarding IPv4 and letting iptables see bridged traffic

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

# Verify that the br_netfilter, overlay modules are loaded by running the following commands:
lsmod | grep br_netfilter
lsmod | grep overlay

# Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward system variables are set to 1 in your sysctl config by running the following command:
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

4) Install container runtime

```
curl -LO https://github.com/containerd/containerd/releases/download/v1.7.14/containerd-1.7.14-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-1.7.14-linux-amd64.tar.gz
curl -LO https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mkdir -p /usr/local/lib/systemd/system/
sudo mv containerd.service /usr/local/lib/systemd/system/
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl daemon-reload
sudo systemctl enable --now containerd

# Check that containerd service is up and running
systemctl status containerd
```

5) Install runc

```
curl -LO https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```

6) install cni plugin

```
curl -LO https://github.com/containernetworking/plugins/releases/download/v1.5.0/cni-plugins-linux-amd64-v1.5.0.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.5.0.tgz
```

7) Install kubeadm, kubelet and kubectl

```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet=1.29.6-1.1 kubeadm=1.29.6-1.1 kubectl=1.29.6-1.1 --allow-downgrades --allow-change-held-packages
sudo apt-mark hold kubelet kubeadm kubectl
---
# 1. Update package metadata
sudo yum update -y

# 2. Install required tools
sudo yum install -y curl ca-certificates gnupg2

# 3. Add Kubernetes YUM repository for v1.29
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
EOF

# 4. Update YUM cache
sudo yum makecache -y

# 5. Install specific Kubernetes versions
sudo yum install -y kubelet-1.29.6 kubeadm-1.29.6 kubectl-1.29.6

# 6. Enable kubelet
sudo systemctl enable --now kubelet

# 7. Optional: Prevent automatic upgrades (not via yum-mark, use version lock plugin)
sudo yum install -y yum-plugin-versionlock
sudo yum versionlock add kubelet kubeadm kubectl
----

kubeadm version
kubelet --version
kubectl version --client
```
>Note: The reason we are installing 1.29, so that in one of the later task, we can upgrade the cluster to 1.30

8) Configure `crictl` to work with `containerd`

`sudo crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock`

9) initialize control plane

```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=172.31.89.68 --node-name master
-------
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=172.31.82.105 \
  --node-name master \
  --ignore-preflight-errors=NumCPU,Mem
```
>Note: Copy the copy to the notepad that was generated after the init command completion, we will use that later.

10) Prepare `kubeconfig`

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
11) Install calico 

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml

curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml -O

kubectl apply -f custom-resources.yaml
```

### Perform the below steps on both the worker nodes

- Perform steps 1-8 on both the nodes
- Run the command generated in step 9 on the Master node which is similar to below

```
sudo kubeadm join 172.31.71.210:6443 --token xxxxx --discovery-token-ca-cert-hash sha256:xxx
```
- If you forgot to copy the command, you can execute below command on master node to generate the join command again

```
kubeadm token create --print-join-command
```
##### In master Node #####
### copy the content in master node from below location in master node and paste in worker node
[root@ip-172-31-82-105 ~]# cat $HOME/.kube/config

##### In worker node ######
##In worker node if we are not able connect to kubectl follow below steps in worker node

[root@ip-172-31-89-221 ~]# mkdir -p $HOME/.kube
[root@ip-172-31-89-221 ~]#
[root@ip-172-31-89-221 ~]# ls -ltrh -a
total 102M
-rw-r--r--.  1 root root  129 Feb  2  2023 .tcshrc
-rw-r--r--.  1 root root  100 Feb  2  2023 .cshrc
-rw-r--r--.  1 root root  429 Feb  2  2023 .bashrc
-rw-r--r--.  1 root root  141 Feb  2  2023 .bash_profile
-rw-r--r--.  1 root root   18 Feb  2  2023 .bash_logout
dr-xr-xr-x. 18 root root  237 Jun  7 06:55 ..
drwx------.  2 root root   29 Jun 12 10:31 .ssh
-rw-r--r--.  1 root root  46M Jun 12 11:26 containerd-1.7.14-linux-amd64.tar.gz
-rw-r--r--.  1 root root  11M Jun 12 11:27 runc.amd64
-rw-r--r--.  1 root root  46M Jun 12 11:27 cni-plugins-linux-amd64-v1.5.0.tgz
-rw-------.  1 root root 3.0K Jun 12 11:35 .bash_history
drwxr-xr-x.  2 root root    6 Jun 12 13:01 .kube
dr-xr-x---.  4 root root  241 Jun 12 13:01 .

[root@ip-172-31-89-221 ~]# cp -p /etc/kubernetes/kubelet.conf $HOME/.kube/
[root@ip-172-31-89-221 ~]# cd /root/.kube/
[root@ip-172-31-89-221 .kube]# ls -ltrh
total 4.0K
-rw-------. 1 root root 2.0K Jun 12 11:42 kubelet.conf
[root@ip-172-31-89-221 .kube]#
##### In worker node ######
[root@ip-172-31-89-221 .kube]# vi config
[root@ip-172-31-89-221 .kube]# kubectl get pods

## Validation

If all the above steps were completed, you should be able to run `kubectl get nodes` on the master node, and it should return all the 3 nodes in ready status.

Also, make sure all the pods are up and running by using the command as below:
` kubectl get pods -A`

>If your Calico-node pods are not healthy, please perform the below steps:

- Disabled source/destination checks for master and worker nodes too.
- Configure Security group rules, Bidirectional, all hosts,TCP 179(Attach it to master and worker nodes)
- Update the ds using the command:
`kubectl set env daemonset/calico-node -n calico-system IP_AUTODETECTION_METHOD=interface=ens5`
Where ens5 is your default interface, you can confirm by running `ifconfig` on all the machines
- IP_AUTODETECTION_METHOD  is set to first-found to let Calico automatically select the appropriate interface on each node.
- Wait for some time or delete the calico-node pods and it should be up and running.
- If you are still facing the issue, you can follow the below workaround

- Install Calico CNI addon using manifest instead of Operator and CR, and all calico pods will be up and running 
`kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml`

This is not the latest version of calico though(v.3.25). This deploys CNI in kube-system NS. 
