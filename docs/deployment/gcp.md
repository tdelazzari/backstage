---
id: gcp
title: Deploying to Google Cloud
sidebar_label: Google Cloud
description: Deploying Backstage to Google Cloud using GKE or Cloud Run
---

This guide explains how to deploy Backstage to [Google Cloud Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine) or [Cloud Run](https://cloud.google.com/run).
GKE is a service provided by Google Cloud that simplifies the deployment, management, and scaling of containerized applications using Kubernetes.
Cloud Run is a more streamlined, serverless option that is also suitable for Backstage.

You will need access to a Google Cloud project with the required permissions to create a GKE cluster or a Cloud Run service.
This guide will also use the following Google Cloud services, so make sure you enabled them in your project:
- [Articact Registry](https://cloud.google.com/artifact-registry) to store the Backstage container image.
- [Cloud SQL](https://cloud.google.com/sql) to create the Backstage database.
- [Secret Manager](https://cloud.google.com/security/products/secret-manager) to store the database password.
- [Cloud Shell](https://cloud.google.com/shell) to run Shell commands such as `kubectl`.

## Push the Backstage container image to Artifact Registry

Follow the [Getting Started](https://backstage.io/docs/getting-started/) guide to install Backstage on your machine.

Edit the Backstage `app-config.yaml` with the following database entry:

```yaml
database:
  client: pg
  connection:
    host: localhost
    port: 5432
    user: postgres
    password: ${POSTGRES_PASSWORD}
```

Install the Google [Cloud CLI](https://cloud.google.com/sdk/docs/install-sdk) on your machine and [configure Docker](https://cloud.google.com/artifact-registry/docs/docker/pushing-and-pulling#cred-helper) to be able to push images to Artifact Registry.
Replace `PROJECT_ID` with your Google Cloud Project ID in the following commands in order to build and push your container image to Artifact Registry:

```bash
yarn build-image --tag backstage:1.0.0
docker tag backstage gcr.io/PROJECT_ID/backstage:1.0.0
docker push gcr.io/PROJECT_ID/backstage:1.0.0
```

## Create the Backstage database in Cloud SQL

Navigate to the [Google Cloud console](https://console.cloud.google.com) and create a new Cloud SQL instance. You can tweak the configuration depending on your needs. Here is the configuration used for this guide:
- Engine and version: `PostgreSQL 15`
- Edition: `Enterprise`
- Region: `us-east1` (single zone)
- Machine configuration: `2 vCPU, 16 GB`

Save your Cloud SQL password in [Secret Manager](https://console.cloud.google.com/security/secret-manager) under the name `POSTGRES_PASSWORD`.

## Create a Service Account for Backstage

Navigate to the [Google Cloud console](https://console.cloud.google.com) and create a service account that you can call `sa-backstage`. It will be used by your Backstage app to access the database and also download the container image from Artifact Registry. You can assign these roles to the service account:
- Artifact Registry Reader
- Cloud SQL Editor

Using Cloud Shell, grant the service account access to the database password you stored in Secret Manager:

```shell
gcloud secrets add-iam-policy-binding POSTGRES_PASSWORD \
    --member=serviceAccount:sa-backstage@PROJECT_ID.iam.gserviceaccount.com \
    --role='roles/secretmanager.secretAccessor'
```

## OPTION 1: Deployment on GKE

Navigate to the [Google Cloud console](https://console.cloud.google.com) and create a GKE cluster. You can tweak the configuration depending on your needs. Here is the configuration used for this guide:
- Mode: `Standard` (Autopilot works too)
- Name: `cluster-backstage`
- Location: `us-east1-c` (zonal)
- Nodes: 3 nodes using `e2-standard-2` machine type

Once the cluster is up and running, use Cloud Shell to [create the Backstage namespace](https://backstage.io/docs/deployment/k8s#creating-a-namespace).

### Set up Workload Identity

[Workload identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) federation for GKE allows workloads in your GKE clusters to impersonate Identity and Access Management (IAM) service accounts to access Google Cloud services.

Using Cloud Shell, create the Kubernetes service account and enable the IAM binding between the Kubernetes and Google Cloud service accounts.
Replace `PROJECT_ID` with your Google Cloud Project ID in the following commands:

```shell
kubectl create serviceaccount ksa-backstage --namespace backstage

gcloud iam service-accounts add-iam-policy-binding \
    sa-backstage@PROJECT_ID.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:PROJECT_ID.svc.id.goog[backstage/ksa-backstage]"

kubectl annotate serviceaccount ksa-backstage \
    --namespace backstage \
    iam.gke.io/gcp-service-account=sa-backstage@PROJECT_ID.iam.gserviceaccount.com
```

### Deployment file

In Cloud Shell, create the file `backstage-deployment.yaml`. Copy and paste the block of text below, replacing `PROJECT_ID` and `CLOUD_SQL_REGION` with your values:

```yaml
# backstage-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backstage
  namespace: backstage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backstage
  template:
    metadata:
      labels:
        app: backstage
    spec:
      serviceAccountName: ksa-backstage
      containers:
        - name: backstage
          image: gcr.io/PROJECT_ID/backstage:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 7007
          env:
            - name: POSTGRES_PASSWORD
            value: "POSTGRES_PASSWORD"
        - name: cloud-sql-proxy
          image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:latest
          args:
            - "--structured-logs"
            - "--port=5432"
            - "PROJECT_ID:CLOUD_SQL_REGION:pg-backstage"
          securityContext:
            runAsNonRoot: true
          resources:
            requests:
              memory: "2Gi"
              cpu:    "1"
```

This deployment spec will also load the [Cloud SQL Auth Proxy in a sidecar pattern](https://cloud.google.com/sql/docs/mysql/connect-kubernetes-engine#run_the_in_a_sidecar_pattern).

In Cloud Shell, run this command to apply the deployment:

```shell
kubectl apply -f backstage-deployment.yaml
```

Finally, you can create the Backstage Kubernetes [service](https://backstage.io/docs/deployment/k8s/#creating-a-backstage-service) and even an [Ingress resource](https://cloud.google.com/kubernetes-engine/docs/tutorials/http-balancer#creating_an_ingress_resource) (Load Balancer) if you want to provide internal and/or external access to this Backstage installation.

## OPTION 2: Deployment on Cloud Run

GKE provides a good level of abstraction if you still want to configure and tweak your own Kubernetes cluster, for example: changing machine types, etc. But if instead you are looking for a more Serverless solution, you should try running Backstage on [Cloud Run](https://cloud.google.com/run).

Navigate to Cloud Run in the [Google Cloud console](https://console.cloud.google.com), and create a new Backstage service. Point to the gcr.io container image in your Artifact Registry. 
Here is the configuration used for this guide:
- Region: `us-east1`
- Container port: `7007`
- Cloud SQL connection: select your Cloud SQL instance in the list
- Security & service account: `sa-backstage`
- Inject `POSTGRES_PASSWORD` as environment variable from Secret Manager

Load the Cloud SQL Auth Proxy as a sidecar. In Cloud Shell, create a new file `backstage-cloudrun.yaml`. Copy and paste the block of text below, replacing `PROJECT_ID` and `CLOUD_SQL_REGION` with your values:

```yaml
# backstage-cloudrun.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  annotations:
     run.googleapis.com/launch-stage: BETA
  name: backstage
spec:
  template:
    metadata:
      annotations:
        run.googleapis.com/execution-environment: gen1 #or gen2
    spec:
      containers:
      - image: gcr.io/PROJECT_ID/backstage:1.0.0
        ports:
          - containerPort: 7007
      - image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:latest
        args:
            - "--structured-logs"
            - "--port=5432"
            - "PROJECT_ID:CLOUD_SQL_REGION:pg-backstage"
```

Update your Cloud Run service using this command:

```
gcloud run services replace backstage-cloudrun.yaml
```

You should now be able to access Backstage via the Cloud Run URL provided.
