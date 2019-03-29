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
    --custom-cpu 4 \
    --custom-memory 5 \
    --network k8s \
    --can-ip-forward

gcloud compute instances create k8s-worker \
    --zone us-central1-b \
    --image-family ubuntu-1604-lts \
    --image-project ubuntu-os-cloud \
    --custom-cpu 4 \
    --custom-memory 5 \
    --network k8s \
    --can-ip-forward
```

5. In the GCP portal ssh to both the Master and the Worker vm's and run the following commands on both vms
```
sudo apt-get update
sudo apt-get install -y docker.io apt-transport-https curl jq nmap iproute2
```

6. Install kubeadm, kubelet, and kubectl
```
sudo su
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat > /etc/apt/sources.list.d/kubernetes.list <<EOF
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

7. On the master node start the cluster
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

8. Join the worker node to the master node you've just started. You will have goten the token and command to join. See my example below:

```
FROM THE WORKER NODE:
kubeadm join 10.128.0.4:6443 --token fsjsrk.mb2013vdg8w6nvjv \
    --discovery-token-ca-cert-hash sha256:330f082be3b755b4b26322e993cf3fc97ff2e3a01380c5c655321f23cec8e985
```

You should see something like the following:
```
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.
```
Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
