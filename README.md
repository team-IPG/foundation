# foundation

foundational artifacts, scripts and design materials for the platform

# GCP Setup

1. Define project and service account names

```bash
export PROJECT_ID=team-ipg-wf
export ACCOUNT_NAME=ipg-service-account
```

2. Authenticate gcloud CLI

```bash
gcloud auth login
```

4.  Create a new Google Cloud Project (or select an existing project).

```bash
gcloud projects create $PROJECT_ID
gcloud config set project $PROJECT_ID
```

5. Enable Cloud Build, Cloud Run, Container Registry, Text-to-Speech APIs, Secrets mgmt

```bash
gcloud services enable cloudbuild.googleapis.com run.googleapis.com 
gcloud services enable containerregistry.googleapis.com texttospeech.googleapis.com
gcloud services enable secretmanager.googleapis.com

```

6. Create a Google Cloud service account

```bash
gcloud iam service-accounts create $ACCOUNT_NAME \
  --description="Cloud Run deploy account" \
  --display-name="Cloud-Run-Deploy"
```

7. Create secret for DB service account

```bash
echo -n "cloud-sql-db-password" | \
  gcloud secrets create employee-db-pwd --data-file=- --replication-policy=automatic
  
Created version [1] of the secret [employee-db-pwd]
```

8. Add the following [Cloud IAM roles][roles] to your service account:

    - `Cloud Run Admin` - allows for the creation of new Cloud Run services
    - `Service Account User` -  required to deploy to Cloud Run as service account
    - `Storage Admin` - allow push to Google Container Registry (this grants project level access, but recommend reducing this scope to [bucket level permissions](https://cloud.google.com/container-registry/docs/access-control#grant).)
    - `PubSub Editor` -  allow topic and subscripton maintenance
    - `Speech-to-Text` - convert names into audio
    - `SecretAccessor` - access any secrets created in the Project
    
```bash
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

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/secretmanager.secretAccessor

```
9. Generate a JSON service account key  for the service account.

```bash
gcloud iam service-accounts keys create key.json \
    --iam-account $ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com
    
# creates a key.json file

# Place in your user home directory: eg. /Users/john/key.json
```    

10. Set environment variable (local development)

```bash

export GOOGLE_APPLICATION_CREDENTIALS=/Users/john/key.json

```
# CICD Process using GitHub Action Workflow

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

- `GCP_SA_KEY`: the downloaded service account key json

## Example Usage

```yaml
name: Build and Deploy to Cloud Run

# Perform CICD only on push to main branch
on:
  push:
    branches:
      - main

env:
  PROJECT_ID: ${{ secrets.GCP_PROJECT }}
  REGION: us-central1
  SERVICE: name-svc
  ARTIFACT: name-svc.jar
  MIN_INSTANCES: 2
  MEMORY: 512Mi

jobs:
  setup-build-deploy:
    name: Setup, Build, and Deploy
    runs-on: ubuntu-latest

    # Add "id-token" with the intended permissions.
    permissions:
      contents: 'write'
      id-token: 'write'

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Set up JDK 17
        uses: actions/setup-java@v2
        with:
          java-version: 17
          distribution: 'adopt'

      # Setup GCP Auth (used by app unit/integration tests and GCP build/deploy steps
      - id: 'auth'
        uses: 'google-github-actions/auth@v0'
        with:
          credentials_json: '${{ secrets.GCP_SA_KEY }}'

      # Use the gradle-build-action which caches by default
      - name: Setup Gradle
        uses: gradle/gradle-build-action@v2

      - name: Assemble JAR
        run: ./gradlew build

      - name: Generate JaCoCo Badge
        uses: cicirello/jacoco-badge-generator@v2
        with:
          jacoco-csv-file: build/reports/jacoco/test/jacocoTestReport.csv

      - name: Commit and push JaCoCo badge (if changed)
        uses: EndBug/add-and-commit@v7
        with:
          default_author: github_actions
          message: 'commit badge'
          add: '*.svg'

      - name: Upload JaCoCo coverage report
        uses: actions/upload-artifact@v2
        with:
          name: jacoco-report
          path: build/reports/jacoco/test/html/

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
        # --quiet
```
