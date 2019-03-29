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
Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

9. Configure kubectl to connect to the newly created cluster
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

10. Check that kubectl get nodes works from the Master node
```
kubectl get nodes 
NAME         STATUS     ROLES    AGE   VERSION
k8s-master   NotReady   master   15m   v1.14.0
k8s-worker   NotReady   <none>   12m   v1.14.0
```

### Configuring a CNI plug-in

11. Check the subnets for both the Master and Worker
```
root@k8s-master:~# kubectl describe node k8s-master | grep PodCIDR
PodCIDR:                     10.244.0.0/24
root@k8s-master:~# kubectl describe node k8s-worker | grep PodCIDR
PodCIDR:                     10.244.1.0/24
```

12. Create CNI plugin configuration on BOTH the Master and Worker vms
```
mkdir -p /etc/cni/net.d
sudo vim /etc/cni/net.d/10-bash-cni-plugin.conf
{
        "cniVersion": "0.3.1",
        "name": "mynet",
        "type": "bash-cni",
        "network": "10.244.0.0/16",  
        "subnet": "<node-cidr-range>" <--- replace this with the CIDR block you got the the output of step 11
}
```
Do this for both the Master and Worker vms

13. Create a network bridge
The network bridge is a special device that aggregates network packets from multiple network interfaces. 

```
root@k8s-master:/etc# sudo brctl addbr cni0
root@k8s-master:/etc# sudo ip link set cni0 up
root@k8s-master:/etc# sudo ip addr add 10.244.0.1/24 dev cni0

root@k8s-worker:# sudo brctl addbr cni0
root@k8s-worker:# sudo ip link set cni0 up
root@k8s-worker:# sudo ip addr add 10.244.1.1/24 dev cni0

root@k8s-worker:# ip route | grep cni0
10.244.1.0/24 dev cni0  proto kernel  scope link  src 10.244.1.1 

root@k8s-master:/etc# ip route | grep cni0
10.244.0.0/24 dev cni0  proto kernel  scope link  src 10.244.0.1 
```

14. Create the cni plug-in 
```
On both the master and worker nodes add the following script 
https://github.com/s-matyukevich/bash-cni-plugin/blob/master/01_gcp/bash-cni
Add the contents of this script on the Master and the Worker
vim /opt/cni/bin/bash-cni
sudo chmod +x /opt/cni/bin/bash-cni
```

15. Testing the plugin we've just created
After adding the bash-cni file you should will now see that the Master and Worker are both in the ready state
```
kubectl get nodes 
NAME         STATUS   ROLES    AGE    VERSION
k8s-master   Ready    master   153m   v1.14.0
k8s-worker   Ready    <none>   149m   v1.14.0
```
We want to test cross-node container communication, so we need to deploy some pods on the master, as well as on the worker.

Note:
```
By default, the scheduler will not put any pods on the master node, because it is “tainted.” But we want to test cross-node container communication, so we need to deploy some pods on the master, as well as on the worker. The taint can be removed using the following command.

FROM THE MASTER NODE:
root@k8s-master:/etc# kubectl taint nodes k8s-master node-role.kubernetes.io/master-
node/k8s-master untainted

Now the scheduler will put pods on the master node.
```

16. Create a test deployment to test the CNI plugin
```
From the master:
kubectl apply -f https://raw.githubusercontent.com/s-matyukevich/bash-cni-plugin/master/01_gcp/test-deployment.yml

Once the containers have started you should see something like the following output:
kubectl describe pod | grep IP
IP:                 10.244.0.3
IP:                 10.244.1.1
IP:                 10.244.0.2
IP:                 10.244.1.0

```

17. kubectl exec into the bash-master pod and run Ping test
```
kubectl exec -it bash-master bash

Can ping itself
root@bash-master:/# ping 10.244.0.3
PING 10.244.0.3 (10.244.0.3) 56(84) bytes of data.
64 bytes from 10.244.0.3: icmp_seq=1 ttl=64 time=0.027 ms
64 bytes from 10.244.0.3: icmp_seq=2 ttl=64 time=0.066 ms

Can’t ping a container on the same host
root@bash-master:/# ping 10.244.0.2

Can't ping a container on a different host
root@bash-master:/# ping 10.244.1.1
PING 10.244.1.1 (10.244.1.1) 56(84) bytes of data.

```

18. Fixing container-to-container communication

```
In order to fix the issue, we need to apply additional forwarding rules that will allow to freely forward traffic inside the whole pod CIDR range. You should execute the two commands below on both Master and Worker VMs. This should fix the issues with communication between containers located at the same host.

sudo iptables -t filter -A FORWARD -s 10.244.0.0/16 -j ACCEPT
sudo iptables -t filter -A FORWARD -d 10.244.0.0/16 -j ACCEPT
```

19. Fixing external access using NAT so that the router doesn't drop packets 

```
Run the following on the master
sudo iptables -t nat -A POSTROUTING -s 10.244.0.0/24 ! -o cni0 -j MASQUERADE
Run the following on the worker
sudo iptables -t nat -A POSTROUTING -s 10.244.1.0/24 ! -o cni0 -j MASQUERADE
```
