# Metrics Server setup for Kubernetes 

## Container runtime 
#### Install packages to allow apt to use a repository over HTTPS

``` 
sudo apt-get update && apt-get install -y \ apt-transport-https ca-certificates curl software-properties-common gnupg2 
```
#### Add Docker's official GPG key:
```
wget -c https://dl.google.com/go/go1.15.linux-amd64.tar.gz -O - | sudo tar -xz -C /usr/local 
```
```
export PATH=$PATH:/usr/local/go/bin
```
```
source ~/.profile
```

#### To install docker
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
```
sudo apt update
```
```
sudo apt install docker.io
```
```
sudo systemctl start docker.
```
```
sudo groupadd docker
```
```
sudo usermod -aG docker $USER
```

#### Restart Docker
```
sudo systemctl daemon-reload
```
```
sudo systemctl restart docker
```
```
sudo systemctl enable docker
```

#### Reboot the VM to apply changes 
```
sudo reboot
```
Once you have your system rebooted continue with following steps

```
sudo modprobe overlay
```
```
sudo modprobe br_netfilter
```
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```
```
sudo sysctl --system
```
```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
```
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```
```
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```
```
sudo apt-get update
```
```
sudo apt-get install -y kubelet kubeadm kubectl
```
```
sudo systemctl daemon-reload
```
```
sudo systemctl restart kubelet
```
```
sudo apt-get update && apt-get upgrade
```
## kubeadm Setup
#### only for master node
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
SAVE THE TOKEN
```
mkdir -p $HOME/.kube
```
```
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```
```
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
```
sudo kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
or use CALICO if this does not work
Download the flannel networking manifest for the Kubernetes API datastore
```
curl https://docs.projectcalico.org/manifests/canal.yaml -O
```
If you are using pod CIDR 10.244.0.0/16, skip to the next step. If you are using a different pod CIDR with kubeadm, no 
changes are required - Calico will automatically detect the CIDR based on the running configuration. For other platforms,
 make sure you uncomment the CALICO_IPV4POOL_CIDR variable in the manifest and set it to the same value as your chosen pod CIDR.
```
kubectl apply -f canal.yaml
```
HELPFUL FOR MANAGING SERVER SERVICES
```
sudo systemctl daemon-reload
```
```
sudo systemctl restart kubelet
```
```
sudo systemctl enable kubelet 
```
PASS THE SAVED TOKEN ON ALL NODES and RUN BELOW TWO COMMANDS
```
sudo systemctl restart kubelet
```
```
sudo systemctl enable kubelet 
```
FOR INSTALLING METRICS SERVER
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.7/components.yaml
```
CHECK IF THE DEPLOYMENT, SERVICES are UP and RUNNING. IF NOT, DEBUG and PROCEED THEN TO NEXT STEP
NOW WE EDIT THE DEPLOYMENT
```
kubectl -n kube-system edit deployment.apps metrics-server
```
ENTER THE INSERT/EDIT MODE
ADD BELOW COMMANDS IN THE FILE 
```yaml
command:
- /metrics-server
- --kubelet-insecure-tls
- --kubelet-preferred-address-types=InternalIP
```
```yaml
     k8s-app: metrics-server
      name: metrics-server
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        command:
        - /metrics-server
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalIP
        image: k8s.gcr.io/metrics-server/metrics-server:v0.3.7
        imagePullPolicy: IfNotPresent
        name: metrics-server
        ports:
        - containerPort: 4443
          name: main-port
          protocol: TCP
```
SAVE

WAIT FOR THE DEPLOYMENT TO ACTION 

*ON MASTER NODE*
```
sudo systemctl daemon-reload
```
```
sudo systemctl restart kubelet
```
```
sudo systemctl enable kubelet 
```
*ON WORKER NODES*
```
sudo systemctl restart kubelet
```
```
sudo systemctl enable kubelet 
```
TO CHECK THE METRICS
```
kubectl top node
```
If you get following errors 

```Error from server (ServiceUnavailable): the server is currently unable to handle the request (get nodes.metrics.k8s.io)```

TO-DO: restart and enable kubelet on master/worker

OR 

```error: metrics not available yet```

TO-DO: restart and enable kubelet on alk worker nodes 

CHECK THE METRICS AGAIN
```
kubectl top node
```

SAMPLE OUTPUT
```
shivaniuga@master:~$ kubectl top node
NAME     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
master   161m         2%     1014Mi          3%        
node1    31m          0%     489Mi           1%        
node2    33m          0%     262Mi           0% 
```

Notes:- 
Enable/restart Kubelet for every change you make for the kubeadm or deploy anything for metrics
```
sudo systemctl restart kubelet
sudo systemctl enable kubelet 
```

## References: 
1. https://kubernetes.io/docs/setup/production-environment/
2. https://github.com/kubernetes-sigs/metrics-server
