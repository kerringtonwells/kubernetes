### Create CNI from scratch with GCP and Kubernetes 

1. Autenticate with using Gcloud and set project
```
  gcloud auth login
  gcloud config set project PROJECT_ID
```

2. Create k8's network in targeted gcloud project
```
  gcloud compute networks create k8s
```
