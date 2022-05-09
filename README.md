# foundation
foundational artifacts and design materials for the platform


# gcp setup

```
export PROJECT_ID=ipg-team
export ACCOUNT_NAME=ipg-service-account
```

```
gcloud services enable cloudbuild.googleapis.com run.googleapis.com containerregistry.googleapis.com
```

```
gcloud iam service-accounts create $ACCOUNT_NAME \
  --description="Cloud Run deploy account" \
  --display-name="Cloud-Run-Deploy"
```

```
# Confirm service account inherits these roles by default

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/run.admin

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/storage.admin

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/iam.serviceAccountUser
  ```
  
```
gcloud iam service-accounts keys create key.json \
    --iam-account $ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com
```    
