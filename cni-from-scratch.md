### Create CNI from scratch with GCP and Kubernetes 

1. Autenticate with using Gcloud and set project
```
  gcloud auth login
  gcloud config set project PROJECT_ID
```

2. Create k8's network in targeted gcloud project
```
gcloud compute networks create k8s

Sample output
NAME  SUBNET_MODE  BGP_ROUTING_MODE  IPV4_RANGE  GATEWAY_IPV4
k8s   AUTO         REGIONAL

Instances on this network will not be reachable until firewall rules
are created. As an example, you can allow all internal traffic between
instances as well as SSH, RDP, and ICMP by running:

$ gcloud compute firewall-rules create <FIREWALL_NAME> --network k8s --allow tcp,udp,icmp --source-ranges <IP_RANGE>
$ gcloud compute firewall-rules create <FIREWALL_NAME> --network k8s --allow tcp:22,tcp:3389,icmp
```


3. To reach your network from the outside create the following firewall rule
```
gcloud compute firewall-rules create k8s-allow-all \
    --network k8s \
    --action allow \
    --direction ingress \
    --rules all \
    --source-ranges 0.0.0.0/0 \
    --priority 1000
```

4. Create Kubernetes master and worker 
```
gcloud compute instances create k8s-master \
    --zone us-central1-b \
    --image-family ubuntu-1604-lts \
    --image-project ubuntu-os-cloud \
    --network k8s \
    --can-ip-forward

gcloud compute instances create k8s-worker \
    --zone us-central1-b \
    --image-family ubuntu-1604-lts \
    --image-project ubuntu-os-cloud \
    --network k8s \
    --can-ip-forward
```

5. In the GCP portal ssh to both the client and the worker vm's and run the following commands on both vms
```
sudo apt-get update
sudo apt-get install -y docker.io apt-transport-https curl jq nmap iproute2
```

6. Install kubeadm, kubelet, and kubectl
sudo su
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat > /etc/apt/sources.list.d/kubernetes.list <<EOF
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF

7. On the master node start the cluster
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
