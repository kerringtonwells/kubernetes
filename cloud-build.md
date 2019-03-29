## Google Cloud Build

1. In Cloud Shell, enable the required APIs.

```
gcloud services enable container.googleapis.com \
    cloudbuild.googleapis.com \
    sourcerepo.googleapis.com \
    containeranalysis.googleapis.com
 ```  

2. In Cloud Shell, create a GKE cluster that you will use to deploy the sample application of this tutorial.

 ``` 
 gcloud container clusters create hello-cloudbuild \
    --num-nodes 1 --zone us-central1-b
 ``` 
 
### Exercise 2: Create the Git repositories in Cloud Source Repositories

1. In Cloud Shell, create the two Git repositories.

```
gcloud source repos create hello-cloudbuild-app
gcloud source repos create hello-cloudbuild-env
````

2. Clone the sample code from GitHub

```
cd ~
git clone https://github.com/GoogleCloudPlatform/gke-gitops-tutorial-cloudbuild \
    hello-cloudbuild-app
```

3. Configure Cloud Source Repositories as a remote.

```
cd ~/hello-cloudbuild-app
PROJECT_ID=$(gcloud config get-value project)
git remote add google \
    "https://source.developers.google.com/p/${PROJECT_ID}/r/hello-cloudbuild-app"
```

### Exercise 3: Create a container image with Cloud Build

1. In Cloud Shell, create a Cloud Build build based on the latest commit with the following command.

```
cd ~/hello-cloudbuild-app
COMMIT_ID="$(git rev-parse --short=7 HEAD)"
gcloud builds submit --tag="gcr.io/${PROJECT_ID}/hello-cloudbuild:${COMMIT_ID}" .
```

2. After the build finishes, verify that your new container image is indeed available in Container Registry


### Exercise 4: Create the continuous integration pipeline

1. Using the code below. Create a file named cloudbuild.yaml
https://github.com/GoogleCloudPlatform/gke-gitops-tutorial-cloudbuild/blob/master/cloudbuild.yaml

2. Open the Triggers page of Cloud Build and go to Triggers

3. Click Create trigger.

4. Select "Cloud Source Repositories" as source and click Continue.

5. Select the hello-cloudbuild-app repository and click Continue.

6. In the "Triggers settings" screen, enter the following parameters:

```
Name: hello-cloudbuild
Branch (regex): master
Build configuration: cloudbuild.yaml
Click Create trigger.
```
7. In Cloud Shell, push the application code to Cloud Source Repositories to trigger the CI pipeline in Cloud Build.

```
cd ~/hello-cloudbuild-app
git push google master
```

8. Open the Cloud Build console.

You should see a build running or having recently finished. You can click on the build to follow its execution and examine its logs.
