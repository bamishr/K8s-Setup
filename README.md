# Kubernetes setup details.

### Contents

- [Installing Kubernetes using installer provided by romana](#installing-kubernetes-using-installer-provided-by-romana)
- [Installing Kubernetes](#installing-kubernetes)
  - [Ubuntu](#ubuntu)
  - [Centos](#centos)
- [Installing Romana using provided containers](#installing-romana-using-provided-containers)
- [Scheduling Pods on Kubernetes master](#scheduling-pods-on-kubernetes-master)
  - [Pre Kubernetes 1.6](#pre-kubernetes-16)
  - [Post Kubernetes 1.6](#post-kubernetes-16)
  - [Retaint master node](#retaint-master-node)
- [Changing/Adding labels to nodes/pods](#changingadding-labels-to-nodespods)
- [Testing if the install is working](#testing-if-the-install-is-working)
- [Change kube proxy to use userspace mode](#change-kube-proxy-to-use-userspace-mode)
- [Bringup and Expose service to external world](#bringup-and-expose-service-to-external-world)
  - [Removing cirros and nginx deployments](#removing-cirros-and-nginx-deployments)
  - [Removing kubernetes install](#removing-kubernetes-install)
  - [Deleting All docker Containers and Images](#deleting-all-docker-containers-and-images)
  - [Copy file form pod to host](#copy-file-form-pod-to-host)
- [Bringup and Expose Kubernetes Dashboard](#bringup-and-expose-kubernetes-dashboard)
- [Installing Romana using installer provided by it partially and rest manually](#installing-romana-using-installer-provided-by-it-partially-and-rest-manually)
- [Some useful commands](#some-useful-commands)
- [Recreate kubeadm join command](#recreate-kubeadm-join-command)

## [Installing Kubernetes using installer provided by romana](#contents)

```bash
git clone https://github.com/romana/romana
cd romana/romana-install
./romana-setup -n test1 -c 3 -s kubeadm  --verbose install

# now ssh into the machines if you want.
ssh -i ~/romana/romana-install/romana_id_rsa ubuntu@<IP Address of the node 1>
ssh -i ~/romana/romana-install/romana_id_rsa ubuntu@<IP Address of the node 2>
ssh -i ~/romana/romana-install/romana_id_rsa ubuntu@<IP Address of the node 3>

# Done, you have working romana installation..
```

## [Installing Kubernetes](#contents)

### [Ubuntu](#contents)
```bash
# On Controller/Master
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF

# Turn swap off, else kubeadm fails, but do not skip pre-flight checks
sudo swapoff -va

sudo apt-get update
sudo apt-get install -y docker.io

# Find and Install a specific version of kubernetes packages as follows if needed:
sudo apt-cache madison kubeadm
sudo apt-get install -y kubelet=1.7.15-00 kubeadm=1.7.15-00 kubectl=1.7.15-00 kubernetes-cni=0.5.1-00
# or else just install the latest one using
sudo apt-get install -y kubeadm

# bootstrap kubernetes
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
# or use a specific version and with flannel which needs pod networking specified
sudo kubeadm init --kubernetes-version v1.7.15 --pod-network-cidr=10.244.0.0/16
# or use a specific version using command below
sudo kubeadm init --kubernetes-version v1.7.15

# now you would get something like this at the end:
# sudo kubeadm join --token=<token> <ip-address:port>
# example:
# sudo kubeadm join --token 0f32b7.c003ad92878711b5 192.168.99.10:6443
# after you get above lines on command prompt, use following
# commands for kubectl to work.
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes -a -o wide --show-labels

# Download and install flannel CNI (Only on Master node once)
wget https://raw.githubusercontent.com/coreos/flannel/v0.11.0/Documentation/kube-flannel.yml
# Change the networking mode from vxlan to host-gw
sed -i 's/vxlan/host-gw/' kube-flannel.yml 
kubectl apply -f kube-flannel.yml

# Download and install kube-router CNI if you don't want to use flannel above (Only on Master node once)
wget https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter.yaml
# Turn off network policies
sed -i 's/--run-firewall=true/--run-firewall=false/' kubeadm-kuberouter.yaml
# OR Turn off network policies and allow communication from a pod that is behind a Service to its own ClusterIP:Port
sed -i 's/--run-firewall=true/--run-firewall=false\n        - --hairpin-mode=true/' kubeadm-kuberouter.yaml
kubectl apply -f kubeadm-kuberouter.yaml

# On Nodes
# All above steps except the kubeadm init step, use kubeadm join below instead.
# Use the kubeadm join command and token from controller here.
sudo kubeadm join --token=<token> <ip-address:port>
```

### [Centos](#contents)
```bash
# On Controller/Master

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

# Turn swap off, else kubeadm fails, but do not skip pre-flight checks
sudo swapoff -va

sudo yum update
sudo yum install -y docker
sudo systemctl enable docker && sudo systemctl start docker
setenforce 0
sudo yum install -y kubelet kubeadm kubectl

# Find and Install a specific version of kubernetes packagesas follows if needed:
sudo yum list kubeadm 
sudo yum install kubeadm-1.7.15-0 kubelet-1.7.15-0 kubectl-1.7.15-0 kubernetes-cni-0.5.1-1
# or else just install the latest one using
sudo yum install -y kubeadm

# Enable and start kubelet
sudo systemctl enable kubelet && sudo systemctl start kubelet

# bootstrap kubernetes
sudo kubeadm init
# or use a specific version and with flannel which needs pod networking specified
sudo kubeadm init --kubernetes-version v1.7.15 --pod-network-cidr=10.244.0.0/16
# or use a specific version using command below
sudo kubeadm init --kubernetes-version v1.7.15
# or with flannel which needs pod networking specified
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# now you would get something like this at the end:
# sudo kubeadm join --token=<token> <ip-address:port>
# example:
# sudo kubeadm join --token 0f32b7.c003ad92878711b5 192.168.99.10:6443
# after you get above lines on command prompt, use following
# commands for kubectl to work.
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes -a -o wide --show-labels

# Download and install flannel CNI (Only on Master node once)
wget https://raw.githubusercontent.com/coreos/flannel/v0.11.0/Documentation/kube-flannel.yml
# Change the networking mode from vxlan to host-gw
sed -i 's/vxlan/host-gw/' kube-flannel.yml 
kubectl apply -f kube-flannel.yml

# On Nodes
# All above steps except the kubeadm init step, use kubeadm join below instead.
# Use the kubeadm join command and token from controller here.
sudo kubeadm join --token=<token> <ip-address:port>
```

## [Installing Romana using provided containers](#contents)

```bash
wget https://raw.githubusercontent.com/romana/romana/master/containerize/specs/romana-kubeadm.yml
# make changes to defaults like cidr's, etc in romana-kubeadm.yml if wanted.
kubectl apply -f romana-kubeadm.yml
```

## [Scheduling Pods on Kubernetes master](#contents)

Removing taint from kubernetes master allows scheduling pods on master.
This is useful when single node is present.

### [Pre Kubernetes 1.6](#contents)

```bash
kubectl taint nodes --all dedicated:NoSchedule-
```

### [Post Kubernetes 1.6](#contents)

```bash
kubectl taint nodes --all node-role.kubernetes.io/master:NoSchedule-
```

### [Retaint Master Node](#contents)

```bash
kubectl taint node <node-name> node-role.kubernetes.io/master=:NoSchedule
```

## [Changing/Adding labels to nodes/pods](#contents)

```bash
# Add label by patching the node yaml.
kubectl patch node <node-name> -p '{"metadata":{"labels":{"node-role.kubernetes.io/master": ""}}}'

# Add label on cli for the node or pod.
kubectl  label nodes/ubuntu com.company.com/node=cluster1
kubectl  label po/nginx-dzpsr com.company.com/app=web

# Example Output:
kubectl get nodes,pods --show-labels
NAME        STATUS    ROLES     AGE       VERSION   LABELS
no/centos   Ready     <none>    3d        v1.9.4    beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=centos
no/ubuntu   Ready     master    3d        v1.9.4    beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,com.company.com/node=cluster1,kubernetes.io/hostname=ubuntu,node-role.kubernetes.io/master=

NAME             READY     STATUS              RESTARTS   AGE         LABELS
po/nginx-5pntv   0/1       ContainerCreating   0          <invalid>   app=nginx
po/nginx-dwf78   0/1       ContainerCreating   0          <invalid>   app=nginx
po/nginx-dzpsr   1/1       Running             0          <invalid>   app=nginx,com.company.com/app=web
po/nginx-vj9s9   1/1       Running             0          <invalid>   app=nginx
```

## [Testing if the install is working](#contents)

```bash
# First try checking the pods and see if all of them
# come up properly and are in running state. You may
# want to wait few minutes before it settles, there
# may be some restarts but eventually it does come up.
kubectl get pods -a -o wide --all-namespaces

# once all pods are up, try installing cirros and see
# if it comes up and if dns works correctly.
kubectl run cirros --image=cirros --replicas=4

# check if the cirros pods are running
kubectl get pods -a -o wide 

# once they are running login into them using
kubectl exec -it <pod-name> /bin/sh

# once inside the pod, use nslookup to test dns
nslookup kubernetes.default
# the result would be as follows:
#Server:    10.96.0.10
#Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
#
#Name:      kubernetes.default
#Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local

nslookup cirros-4036794762-9s46o
#Server:    10.96.0.10
#Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
#
#Name:      cirros-4036794762-9s46o
#Address 1: 100.112.11.131 cirros-4036794762-9s46o

```

## [Change kube proxy to use userspace mode](#contents)

```bash
# Install jq
sudo apt-get install jq # ubuntu
sudo yum install jq # centos

# Patch kube-proxy yaml to include userspace mode
kubectl -n kube-system get ds -l "k8s-app=kube-proxy" -o json | jq ".items[0].spec.template.spec.containers[0].command |= .+ [\"--proxy-mode=userspace\"]" | kubectl apply -f -

# Restart kube-proxy pods, so that the new config can be applied.
kubectl -n kube-system delete pods -l "k8s-app=kube-proxy"
```

## [Bringup and Expose service to external world](#contents)

```bash
# bringup nginx with 2 replicas on port 80.
kubectl run nginx --image=nginx --replicas=2 --port=80

# Check if the pods are up else wait few seconds for them
# to start, when desired and current columns both show
# correct number of replicas,  it means nginx is up and
# running correctly.
kubectl get deployments --all-namespaces

# Now expose the service to outside world.
kubectl expose deployment nginx --port=80 --type=LoadBalancer --external-ip=192.168.99.10

# Check if the configuration was applied correctly, if
# it was, then external IP should be reflected using
# following command.
kubectl get services --all-namespaces -o wide

# Test if nginx is up and running
curl 192.168.99.10:80
#<!DOCTYPE html>
#<html>
#<head>
#<title>Welcome to nginx!</title>
#<snip>...

# You are all set and ready to roll.
```

### [Removing cirros and nginx deployments](#contents)

```bash
kubectl delete deployments nginx cirros
```

### [Removing kubernetes install](#contents)

```bash
# Remove a node from the cluster
kubectl drain <node-name>
kubectl delete node <node-name>

# Allowed drain node to return to cluster
kubectl uncordon <node-name>

# On Controller and all nodes, run following command
# to reset your cluster to what it was before installing
# kubernetes
# BEWARE, YOU WILL LOSE ALL YOUR PODS/SERVICES/DATA/etc.
sudo kubeadm reset
```

### [Deleting All docker Containers and Images](#contents)

```bash
# Removing all docker containers
docker rm $(docker ps -a -q)
# Removing all cleanly exited docker containers
docker rm $(docker ps -a -q -f exited=0)
# Removing all docker images
docker rmi $(docker images -a -q)
# Reclaim space by clearing up docker volume
docker volume prune -f
# List IP address for docker containers
docker ps -q | xargs docker inspect --format '{{ .Id }} - {{ .Name }} - {{ .NetworkSettings.IPAddress }}'
# Bash function for docker ps like IP address details.
function dockerps() {
    docker ps | while read line; do
        if `echo $line | grep -q 'CONTAINER ID'`; then
            echo -e "IP ADDRESS\t$line"
        else
            CID=$(echo $line | awk '{print $1}');
            IP=$(docker inspect -f "{{ .NetworkSettings.IPAddress }}" $CID);
            printf "${IP}\t${line}\n"
        fi
    done;
}
```

### [Copy file form pod to host](#contents)

```bash
sudo docker cp `kubectl get pods -a -o wide --all-namespaces --selector=app=<name> -o   jsonpath='{.items[*].status.containerStatuses[1].containerID}'| cut -d"/" -f3 | cut -c1-30`:/usr/local/bin/<file> /usr/local/bin/<file>
```

## [Bringup and Expose Kubernetes Dashboard](#contents)

```bash
wget https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
kubectl create -f kubernetes-dashboard.yaml
kubectl proxy --port 8001 &
cat << EOF > kubernetes-dashboard-rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
EOF
kubectl apply -f kubernetes-dashboard-rbac.yaml
# Now open the following link for dashboard:
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
```

## [Installing Romana using installer provided by it partially and rest manually](#contents)

```bash
git clone https://github.com/romana/romana
cd romana/romana-install
./romana-setup -n test1 -c 3 -s nostack  --verbose install
ssh -i ~/romana/romana-install/romana_id_rsa ubuntu@<IP Address of the node 1>
ssh -i ~/romana/romana-install/romana_id_rsa ubuntu@<IP Address of the node 2>
ssh -i ~/romana/romana-install/romana_id_rsa ubuntu@<IP Address of the node 3>

# On First node
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y docker.io
sudo apt-get install -y kubeadm
sudo kubeadm init
# or use a specific version using command below
sudo kubeadm init --kubernetes-version v1.7.0
# now you would get something like this at the end:
# sudo kubeadm join --token=<token> <ip-address:port>
# example:
# sudo kubeadm join --token 0f32b7.c003ad92878711b5 192.168.99.10:6443
# this is to be used on other nodes for joining kubernetes cluster.

# Install config for kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes -a -o wide --show-labels

# Install Romana
wget https://raw.githubusercontent.com/romana/romana/master/containerize/specs/romana-kubeadm.yml
kubectl create -f romana-kubeadm.yml

# Remove taints
kubectl taint nodes --all node-role.kubernetes.io/master:NoSchedule-

# On other Nodes
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y docker.io
sudo apt-get install -y kubeadm
# Use the kubeadm join command and token from controller here.
sudo kubeadm join --token=<token> <ip-address:port>
```

## [Some useful commands](#contents)

```bash
# Get Nodes, Pods, Service, etc details.
watch -d kubectl get nodes -a -o wide
watch -d kubectl get pods,svc,rc -a -o wide --all-namespaces
watch -d kubectl get pods,rc,svc,ds,jobs,deploy -a -o wide --all-namespaces

# Delete all services, replication controllers, etc from default namespaces
kubectl get svc,rc,deploy,jobs -o name | xargs kubectl delete

# Delete non running pods.
kubectl get pods --all-namespaces -o json --field-selector="status.phase!=Running" | jq  '.items[] | "kubectl delete pods \(.metadata.name) -n \(.metadata.namespace)"' | xargs -n 1 bash -c

# Remove evicted pods.
kubectl get pods --all-namespaces -ojson | jq -r '.items[] | select(.status.reason!=null) | select(.status.reason | contains("Evicted")) | .metadata.name + " " + .metadata.namespace' | xargs -n2 -l bash -c 'kubectl delete pods $0 --namespace=$1'

# Send bridge packets to iptables for further processing, needed by kubeadm (kubernetes)
echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl net.bridge.bridge-nf-call-iptables=1
sudo sysctl net.bridge.bridge-nf-call-ip6tables=1

# Using Docker command without sudo
sudo groupadd docker
sudo usermod -aG docker $USER
# logout and login, docker should work without sudo

```

## [Recreate kubeadm join command](#contents)

```bash
echo sudo kubeadm join $(ip addr show dev $(awk '$2 == 00000000 { print $1 }' /proc/net/route) | awk '$1 ~ /^inet/ { sub("/.*", "", $2); print $2 }' | head -n 1):6443 --token $(kubeadm token list | head -n 2 | tail -n 1 | cut -d ' ' -f 1) --discovery-token-ca-cert-hash sha256:$(openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //')
```
