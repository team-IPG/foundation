# foundation
foundational artifacts and design materials for the platform



# gcp setup

1. Define project and service account names

```
export PROJECT_ID=team-ipg
export ACCOUNT_NAME=ipg-service-account
```

2. AUthenticate gcloud CLI

```
gcloud auth login
```

4.  Create a new Google Cloud Project (or select an existing project).

```
gcloud projects create $PROJECT_ID
```

5. Enable Cloud Build, Cloud Run and Container Registry APIs

```
gcloud services enable cloudbuild.googleapis.com run.googleapis.com containerregistry.googleapis.com
```

6. Create a Google Cloud service account

```
gcloud iam service-accounts create $ACCOUNT_NAME \
  --description="Cloud Run deploy account" \
  --display-name="Cloud-Run-Deploy"
```

7.  Add the the following [Cloud IAM roles][roles] to your service account:

    - `Cloud Run Admin` - allows for the creation of new Cloud Run services

    - `Service Account User` -  required to deploy to Cloud Run as service account

    - `Storage Admin` - allow push to Google Container Registry (this grants project level access, but recommend reducing this scope to [bucket level permissions](https://cloud.google.com/container-registry/docs/access-control#grant).)
    
    - `PubSub Editor -  allow topic and subscripton maintenance


```
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/run.admin

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/iam.serviceAccountUser


gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/storage.admin
  
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/pubsub.editor

```

8.  Generate a JSON service account key  for the service account.


```
gcloud iam service-accounts keys create key.json \
    --iam-account $ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com
```    


# cloud-run github setup

[github action](https://github.com/google-github-actions/deploy-cloudrun)

## Prerequisites

This action requires:

-   Google Cloud credentials that are authorized to deploy a Cloud Run service.
    See the [Credentials](#credentials) below for more information.

-   [Enable the Cloud Run API](http://console.cloud.google.com/apis/library/run.googleapis.com)

-   This action runs using Node 16. If you are using self-hosted GitHub Actions
    runners, you must use runner version [2.285.0](https://github.com/actions/virtual-environments)
    or newer.
    


Add the following secrets to your repository's secrets:

- `GCP_PROJECT`: Google Cloud project ID

- `GCP_SA_KEY`: the downloaded service account key

## Usage

```yaml
name: Build and Deploy to Cloud Run


on:
  push:
    branches:
    - master
    
env:
  PROJECT_ID: ${{ secrets.RUN_PROJECT }}
  RUN_REGION: us-central1
  SERVICE_NAME: my-service
  
jobs:
  setup-build-deploy:
    name: Setup, Build, and Deploy
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout
      uses: actions/checkout@v2
      
    - id: 'auth'
      uses: 'google-github-actions/auth@v0'
      with:
        credentials_json: '${{ secrets.GCP_SA_KEY }}'

    # Build and push image to Google Container Registry
    - name: Build
      run: |-
        gcloud builds submit \
          --quiet \
          --tag "gcr.io/$PROJECT_ID/$SERVICE_NAME:$GITHUB_SHA"
          
    - name: 'Deploy to Cloud Run'
      uses: 'google-github-actions/deploy-cloudrun@v0'
      with:
        image: "gcr.io/$PROJECT_ID/$SERVICE_NAME:$GITHUB_SHA"
        service: '${{ env.SERVICE_NAME }}'
        region: '${{ env.REGION }}'
        env_vars: 'NAME="Hello World"'
    
    - name: 'Use output'
      run: 'curl "${{ steps.deploy.outputs.url }}"'
      
```
