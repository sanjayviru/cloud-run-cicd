# 02 - Cloud Run + Cloud Build CI/CD

Node/Express app, containerized, auto-deployed to Cloud Run on every push to `main`.

## Project structure
```
02-cloud-run-cicd/
├── server.js
├── package.json
├── public/index.html
├── Dockerfile
├── .dockerignore
├── .gitignore
├── cloudbuild.yaml
└── README.md
```

## Step 1 - Variables and project
```bash
gcloud auth login
export PROJECT_ID=YOUR_PROJECT_ID
export REGION=us-central1
export REPO=web-repo
export SERVICE=web-demo
gcloud config set project $PROJECT_ID
```

## Step 2 - Enable APIs
```bash
gcloud services enable run.googleapis.com cloudbuild.googleapis.com artifactregistry.googleapis.com
```

## Step 3 - Create Artifact Registry repo
```bash
gcloud artifacts repositories create $REPO \
  --repository-format=docker --location=$REGION --description="Docker images"
```

## Step 4 - IAM for the Cloud Build service account
New projects run builds as the default compute service account:
```bash
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
export BUILD_SA=${PROJECT_NUMBER}-compute@developer.gserviceaccount.com

for ROLE in roles/run.admin roles/iam.serviceAccountUser \
            roles/artifactregistry.writer roles/logging.logWriter roles/storage.objectViewer; do
  gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:$BUILD_SA" --role="$ROLE" --condition=None
done
```

## Step 5 - Test locally with Docker
```bash
docker build -t web-demo .
docker run --rm -p 8080:8080 web-demo     # open http://localhost:8080
```

## Step 6 - One manual deploy to verify
```bash
gcloud builds submit --tag $REGION-docker.pkg.dev/$PROJECT_ID/$REPO/$SERVICE:manual .
gcloud run deploy $SERVICE \
  --image $REGION-docker.pkg.dev/$PROJECT_ID/$REPO/$SERVICE:manual \
  --region $REGION --allow-unauthenticated
gcloud run services describe $SERVICE --region $REGION --format='value(status.url)'
```

## Step 7 - Git and GitHub
```bash
git init
git add .
git commit -m "Initial commit: Cloud Run sample app"
git branch -M main
git remote add origin https://github.com/YOUR_USER/cloudrun-cicd-demo.git
git push -u origin main
```

Connect GitHub to Cloud Build (one-time, browser): Console -> Cloud Build -> Triggers -> **Connect repository** -> GitHub -> authorize -> select your repo.

## Step 8 - Create the trigger
```bash
gcloud builds triggers create github \
  --name=deploy-cloudrun-main \
  --repo-name=cloudrun-cicd-demo \
  --repo-owner=YOUR_GITHUB_USER \
  --branch-pattern='^main$' \
  --build-config=cloudbuild.yaml \
  --service-account=projects/$PROJECT_ID/serviceAccounts/$BUILD_SA
```

## Step 9 - Trigger the pipeline
Edit the hero text in `public/index.html`, then:
```bash
git add . && git commit -m "Update hero text" && git push
gcloud builds list --limit=3
gcloud builds log --stream $(gcloud builds list --limit=1 --format='value(id)')
gcloud run services describe $SERVICE --region $REGION --format='value(status.url)'
```
Refresh the URL and you should see the new UI.

## Step 10 - Logs, rollback, cleanup
```bash
gcloud run revisions list --service $SERVICE --region $REGION
gcloud run services update-traffic $SERVICE --region $REGION --to-revisions=REVISION_NAME=100
gcloud run services logs read $SERVICE --region $REGION --limit=50

# cleanup
gcloud builds triggers delete deploy-cloudrun-main
gcloud run services delete $SERVICE --region $REGION
gcloud artifacts repositories delete $REPO --location=$REGION
```

## Troubleshooting
- **Permission denied on deploy**: recheck Step 4 roles.
- **$COMMIT_SHA empty**: it only exists for trigger builds; for manual builds use `--substitutions=COMMIT_SHA=manual`.
- **Trigger creation fails**: connect the GitHub repo in the console first (Step 7).
