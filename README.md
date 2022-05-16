# foundation
foundational artifacts and design materials for the platform



# gcp setup

1. Define project and service account names

```
export PROJECT_ID=team-ipg-wf
export ACCOUNT_NAME=ipg-service-account
```

2. AUthenticate gcloud CLI

```
gcloud auth login
```

4.  Create a new Google Cloud Project (or select an existing project).

```
gcloud projects create $PROJECT_ID

gcloud config set project $PROJECT_ID
```

5. Enable Cloud Build, Cloud Run, Container Registry, Text-to-Speech APIs

```
gcloud services enable cloudbuild.googleapis.com run.googleapis.com containerregistry.googleapis.com

gcloud services enable texttospeech.googleapis.com

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
    
    - `PubSub Editor` -  allow topic and subscripton maintenance

    - `Speech-to-Text` - convert names into audio (TODO)


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

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/cloudbuild.builds.builder

```

8.  Generate a JSON service account key  for the service account.


```
gcloud iam service-accounts keys create key.json \
    --iam-account $ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com
    
creates a key.json file

Place in your user home directory: eg. /Users/john/key.json
```    

9. Set environment variable

```bash

export GOOGLE_APPLICATION_CREDENTIALS=/Users/john/key.json

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
      - main

env:
  PROJECT_ID: ${{ secrets.GCP_PROJECT }}
  REGION: us-central1
  SERVICE: name-svc
  ARTIFACT: name-svc.jar
  MIN_INSTANCES: 1
  MEMORY: 512Mi

jobs:
  setup-build-deploy:
    name: Setup, Build, and Deploy
    runs-on: ubuntu-latest

    # Add "id-token" with the intended permissions.
    permissions:
      contents: 'read'
      id-token: 'write'

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Set up JDK 17
        uses: actions/setup-java@v2
        with:
          java-version: 17
          distribution: 'adopt'

      - name: Assemble with Gradle
        run: ./gradlew assemble

      - id: 'auth'
        uses: 'google-github-actions/auth@v0'
        with:
          credentials_json: '${{ secrets.GCP_SA_KEY }}'

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v0

      - name: Authorize Docker push
        run: gcloud auth configure-docker

      - name: Build and Push Container
        run: |-
          docker build -t gcr.io/${{ env.PROJECT_ID }}/${{ env.SERVICE }}:${{  github.sha }} --build-arg=JAR_FILE=build/libs/${{ env.ARTIFACT }} .
          docker push gcr.io/${{ env.PROJECT_ID }}/${{ env.SERVICE }}:${{  github.sha }}

      - name: Deploy to Cloud Run
        id: deploy
        run: |-
          gcloud run deploy ${{ env.SERVICE }} \
            --region ${{ env.REGION }} \
            --allow-unauthenticated \
            --image gcr.io/${{ env.PROJECT_ID }}/${{ env.SERVICE }}:${{  github.sha }} \
            --memory ${{ env.MEMORY }} \
            --min-instances ${{ env.MIN_INSTANCES }} \
            --platform "managed" \
      
```
