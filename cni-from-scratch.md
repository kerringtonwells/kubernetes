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
    
