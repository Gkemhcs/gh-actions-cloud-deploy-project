# GITHUB-ACTIONS TO CLOUD-DEPLOY 
## THIS PROJECT SHOWS THE INTEGRATION OF GOOGLE CLOUD DEPLOY IN GITHUB ACTIONS TO CREATE RELEASES 

### INTRO:-

TOOLS USED:
      
1. CLOUD DEPLOY:- FOR CONTINOUS DELIVERY OF CONTAINER IMAGES
2. CLOUD ARTIFACT REPOSITORIES:- TO STORE CONTAINER IMAGES
3. GOOGLE KUBERNETES ENGINE:- TO DEPLOY THE CONTAINERIZED APPS 
4. GITHUB ACTIONS :- BUILDING THE CODE  AND CREATING CLOUD-DEPLOY RELEASES ON PUSH TO REPO 
    
    

###  PROJECT INFO:-
 ON PUSHING THE APPCODE CHANGES TO REPOSITORY  THE GITHUB WORKFLOWS BUILTS THE IMAGE AND PUSHES IT TO GOOGLE ARTIFACT REPOSITORY 
       AND THEN CREATE  CLOUD-DEPLOY RELEASES TO  INITIATE CONTINOUS DELIVERY
       PROCESS 
### STEPS TO DEPLOY THE PROJECT:-
1)CLONE THE PROJECT INTO CLOUDSHELL/LOCAL WORKSTATION
```bash
git clone  https://github.com/Gkemhcs/gh-actions-cloud-deploy-gcp.git
cd gh-actions-cloud-deploy-gcp
```

2)FIRST ENABLE THE REQUIRED APIS 
    
 ```bash 
 echo "ENTER YOUR PROJECT_ID"
 read PROJECT_ID
 export PROJECT_ID=$PROJECT_ID
 gcloud config set project $PROJECT_ID
 sed -i "s/GOOGLE_CLOUD_PROJECT_ID/${PROJECT_ID}" .github/workflows/gcp_deploy.yaml
 gcloud  services enable container.googleapis.com compute.googleapis.com \
 clouddeploy.googleapis.com cloudbuild.googleapis.com artifactregistry.googleapis.com 
 ```
 
3)CREATE GKE CLUSTERS 
```bash
gcloud container clusters create cluster-test --zone asia-south2-a \
--workload-pool $PROJECT_ID.svc.id.goog \
--num-nodes 1 \
--machine-type e2-standard-4 

gcloud container clusters create cluster-prod --zone us-central1-a \
--workload-pool $PROJECT_ID.svc.id.goog \
--num-nodes 1 \
--machine-type e2-standard-4 
```
4)CREATE  REQUIRED IAM SERVICE-ACCOUNTS 
```bash
gcloud iam service-accounts create github-sa
gcloud iam service-accounts create deploy-runner
gcloud iam service-accounts create deploy-promoter
```
5)BINDING THE IAM POLICIES TO THE SERVICE ACCOUNTS
```bash

gcloud projects add-iam-policy-binding $PROJECT_ID \
--member "serviceAccount:github-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
--role roles/clouddeploy.releaser
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member "serviceAccount:github-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
--role roles/artifactregistry.createOnPushWriter
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member "serviceAccount:github-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
--role roles/storage.admin
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member "serviceAccount:deploy-runner@${PROJECT_ID}.iam.gserviceaccount.com" \
--role roles/clouddeploy.jobRunner
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member "serviceAccount:deploy-runner@${PROJECT_ID}.iam.gserviceaccount.com" \
--role roles/container.developer
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member "serviceAccount:deploy-promoter@${PROJECT_ID}.iam.gserviceaccount.com" \
--role roles/clouddeploy.operator
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member "serviceAccount:deploy-promoter@${PROJECT_ID}.iam.gserviceaccount.com" \
--role roles/clouddeploy.developer
gcloud iam service-accounts add-iam-policy-binding deploy-runner@$PROJECT_ID.iam.gserviceaccount.com \
--member "serviceAccount:deploy-promoter@${PROJECT_ID}.iam.gserviceaccount.com" \
--role roles/iam.serviceAccountUser
gcloud iam service-accounts add-iam-policy-binding deploy-runner@$PROJECT_ID.iam.gserviceaccount.com \
--member "serviceAccount:github-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
--role roles/iam.serviceAccountUser

```
6)CREATING THE CLOUD DEPLOY DELIVERY PIPELINE TO ENABLE THE CONTINOUS DELIVERY FOR GKE DEPLOYMENTS
```bash
# move to cloud-deploy directory 
cd cloud-deploy 
sed -i "s/PROJECT_ID/${PROJECT_ID}/g" delivery-pipeline.yaml
gcloud deploy apply --file delivery-pipeline.yaml \
--region asia-south2
```
7)CREATING THE GOOGLE ARTIFACT REGISTRY IN ASIA 

```bash
gcloud artifacts repositories create repo-github \
--repository-format docker \
--region asia-south2 \
--labels type=github-images
 ```

8)CREATING THE WORKLOAD IDENTITY POOL AND ADDING GITHUB PROVIDER TO THE POOL FOR AUTHENTICATING,AUTHORIZING THE GITHUB WORKFLOWS
9)**REPLACE THE USER WITH YOUR GITHUB USERNAME AND REPO WITH NAME OF GITHUB REPOSITORY** 

```bash

gcloud iam workload-identity-pools create github-pool  --location global --display-name  GITHUB-POOL
gcloud iam workload-identity-pools providers create-oidc  github --location global \
--workload-identity-pool=github-pool --display-name GITHUB_PROVIDER \
 --issuer-uri="https://token.actions.githubusercontent.com"  \
 --attribute-mapping="google.subject=assertion.sub,attribute.workflow=assertion.workflow,attribute.actor=assertion.actor,attribute.repository=assertion.repository" \
  --attribute-condition='assertion.repository=="USER/REPO"'
```


```bash

export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format "value(projectNumber)")
gcloud iam service-accounts add-iam-policy-binding  github-sa@$PROJECT_ID.iam.gserviceaccount.com  --member "principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github-pool/attribute.workflow/gcp-deploy" --role roles/iam.workloadIdentityUser
```
10) IN GITHUB ADD SECRET NAMED WIF_PROVIDER  IN THE FORMAT **projects/REPLACE_WITH_PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/providers/github**
11) ADDD ANOTHER SECRET NAMED  WIF__SERVICE_ACCOUNT IN THE FORMAT 
**github-sa@PROJECT_ID.iam.gserviceaccount.com**
13) LOGIN TO YOUR GITHUB ACCOUNT TO PUSH THE CODE
```bash
echo "ENTER YOUR REPO URL(HTTPS)"
read REPO_URL
```
```bash
git remote add  target $REPO_URL
git add .
git commit -m "code added"
git push origin main

```
### successfully completed the deployment part ,to experience this use ***kubectl get svc*** to get the service of type loadbalancer named frontend-lb-test in test cluster and frontend-lb-prod in prod cluster ,then again change some code in server.py located in app  folder   and commit the code to see changes to the ui in both services test,prod