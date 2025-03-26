---
author: "Matic Rupnik"
authorLink: matic-rupnik
date: 2025-03-25
title: Generic Postgres backup Helm chart for S3 buckets
tags: [
  "Helm",
  "Kubernetes",
  "Docker",
  "Postgres"
]
---

Here I am back again with another take on a dependency Helm chart that could be useful in some of your cases but especially mine.
It all started with the need to make backups of standalone services that use the PostgreSQL as a back-end database. In my case it was the DefectDojo vulnerability front end that lets you handle the security of your projects.

Speaking a bit more about what the backup is in this case, I am making a simple database dump that will later be stored on a S3 MiniIO instance with the help of a custom script I have packaged into a docker image called mrupnikm/olm-tusky-job.

# How it works

Breaking it down the chart has the following structure:


```
├── charts
│   └── olm-tusky
│       ├── Chart.yaml
│       ├── templates
│       │   ├── cronjob.yaml
│       │   ├── _helpers.tpl
│       │   ├── rbac.yaml
│       │   └── secrets.yaml
│       └── values.yaml
└── tusky-job
    ├── backup.sh
    └── dockerfile
```

1. Tusky Job has the contents of a dockerfile and the backup script used in the job that will be triggering based on time:

```bash
#!/bin/bash
set -e

# Setup MinIO client
mc alias set minio "$MINIO_ENDPOINT" "$MINIO_ACCESS_KEY" "$MINIO_SECRET_KEY" --insecure

# Create backup filename with timestamp
BACKUP_FILE="backup-$(date +%Y%m%d-%H%M%S).sql.gz"

echo "Starting PostgreSQL backup of $PG_DATABASE from $PG_HOST..."
pg_dump -h "$PG_HOST" -p "$PG_PORT" -U "$PG_USER" -d "$PG_DATABASE" | gzip >"$BACKUP_FILE"

echo "Uploading backup to MinIO bucket $MINIO_BUCKET..."
mc cp --insecure "$BACKUP_FILE" "minio/$MINIO_BUCKET/"

# Cleanup
rm "$BACKUP_FILE"

echo "Backup completed successfully and uploaded to MinIO"
```

As you can see the task itself is quite short and simple. All it needs are the `minio-client` binary and access to the `pg_dump` binary to make the backup happen. For now I have passed the needed credentials as environment variables.

2. Values define the behavior of our chart so they look like this for now. I usually use sops encryption if you intend to store them in a repository:
```yaml
image:
  repository: mrupnikm/olm-tusky-job
  tag: latest
  pullPolicy: Always

# Schedule (default: every day at 1 AM UTC)
schedule: "0 1 * * *"

# Existing secret containing credentials
existingSecret: "postgresql-specific"

# Values from your environment
values:
  minioEndpoint:
  minioBucket:
  minioAccessKey:
  minioSecretKey:

# Scaling configuration
scaleTarget:
  deploymentName: "deployment"
  restoreReplicas: 1

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

successfulJobsHistoryLimit: 3
failedJobsHistoryLimit: 1
activeDeadlineSeconds: 600
postgresql:
  database:
  user:

```
Breaking it down it is quote simple here as well. You need to set an image for the job and the cron schedule. Pass in the existing postgresql database password secret name, database and username and the MinIO credentials. Lastly do not forget about the deployment name that needs to be stopped when the backup happens.

3. Templates directory contains the needed Kubernetes manifest files for the chart. Lets tackle the cron job first:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: {{ include "tusky-db-backup.fullname" . }}
  namespace: {{ .Values.namespace }}
  labels:
    {{- include "tusky-db-backup.labels" . | nindent 4 }}
  annotations:
    backup.user: "mrupnikm"
spec:
  schedule: {{ .Values.schedule | quote }}
  successfulJobsHistoryLimit: {{ .Values.successfulJobsHistoryLimit | default 3 }}
  failedJobsHistoryLimit: {{ .Values.failedJobsHistoryLimit | default 1 }}
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      activeDeadlineSeconds: {{ .Values.activeDeadlineSeconds | default 600 }}
      template:
        spec:
          serviceAccountName: {{ include "tusky-db-backup.fullname" . }}
          initContainers:
          - name: scale-down
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              echo "Scaling down deployment {{ .Release.Name }}-{{ .Values.scaleTarget.deploymentName }}..."

              # Store current replicas count
              CURRENT_REPLICAS=$(kubectl get deployment {{ .Release.Name }}-{{ .Values.scaleTarget.deploymentName }} \
                -n {{ .Values.namespace }} -o jsonpath='{.spec.replicas}')

              # Scale down
              kubectl scale deployment {{ .Release.Name }}-{{ .Values.scaleTarget.deploymentName }} \
                --replicas=0 \
                -n {{ .Values.namespace }}

              # Wait for scale down
              echo "Waiting for pods to terminate..."
              kubectl wait --for=delete pod \
                -l app.kubernetes.io/name={{ .Release.Name }}-{{ .Values.scaleTarget.deploymentName }} \
                -n {{ .Values.namespace }} --timeout=60s

              echo "Scale down completed. Original replicas: $CURRENT_REPLICAS"
          containers:
          - name: backup
            image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
            imagePullPolicy: {{ .Values.image.pullPolicy }}
            env:
            - name: PG_HOST
              value: "{{ .Release.Name }}-postgresql"
            - name: PG_PORT
              value: "5432"
            - name: PG_DATABASE
              value: {{ .Values.postgresql.database | quote }}
            - name: PG_USER
              value: {{ .Values.postgresql.user | quote }}
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.existingSecret }}
                  key: postgresql-password
            - name: MINIO_ENDPOINT
              value: {{ .Values.bucket.minioEndpoint | quote }}
            - name: MINIO_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: s3bucket-secret
                  key: accesskey
            - name: MINIO_BUCKET
              value: {{ .Values.bucket.minioBucket | quote }}
            - name: MINIO_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: s3bucket-secret
                  key: secretkey
            resources:
              {{- toYaml .Values.resources | nindent 14 }}
          - name: scale-up
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              echo "Waiting for backup to complete..."
              while [ "$(kubectl get pod $HOSTNAME -n {{ .Values.namespace }} -o jsonpath='{.status.containerStatuses[?(@.name=="backup")].state.terminated}')" = "" ]; do
                sleep 5
              done

              echo "Scaling up deployment..."
              kubectl scale deployment {{ .Release.Name }}-{{ .Values.scaleTarget.deploymentName }} \
                --replicas={{ .Values.scaleTarget.restoreReplicas | default 1 }} \
                -n {{ .Values.namespace }}

              echo "Scale up completed"
            env:
            - name: HOSTNAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          restartPolicy: OnFailure
```

There are 3 containers. One to stop the deployment as to stop writes to the database, one to execute the backup and one to start the main deployment back up again.


# Implementation example

As I have mentioned I will be using Olm-Tusky chart as a dependency in a umbrella chart, besides the main application(DefectDojo).
To achieve this I first needed to get the DefectDojo GIT repository under the latest release tag.

> Note that I used local unpackaged charts for this example

The setup from there is quite simple. I made a Chart.yaml definition and added the 2 dependencies to it:

```yaml
apiVersion: v2
name: tusky-defectdojo
description: A postgresql backup umbrella chart for defectdojo
type: application
version: 0.2.0
appVersion: "2.44.2"

dependencies:
- name: defectdojo
  version: "1.6.176"
  repository: "file://../django-DefectDojo/helm/defectdojo/"
- name: olm-tusky
  version: "0.1.0"
  repository: "file:///../olm-tusky/charts/olm-tusky/."
```

Since this is an umbrella chart, you need to structure your `values.yaml` in the correct way:

```yaml
olm-tusky:
  image: #...
  schedule: #...
  #...
defectdojo:
  #...

```

All tha was needed now was `helm dependency build .` and you are ready to install the chart with `helm install extended-defectdojo -n defectdojo .`.


# The finisher

This creation is by no means finished or perfect but for now it fits my use case for standalone backups of Postgres database to an S3 Bucket. 
The way I stored these secrets could be improved on 2 points. The way I store the access information for the bucket in the values could be changed to reference names with the implementation of a HCP Vault or Infisical on the cluster.
But more importantly The passing of passwords into the image could be made to work with locally mounted files instead of environment variables.

With that out the way. I hope this gives you some inspiration on your own backup solutions and happy Heliming!


# Resources:

- [Github Olm-Tusky](https://github.com/mrupnikm/olm-tusky)
- [Job Docker image](https://hub.docker.com/repository/docker/mrupnikm/olm-tusky-job)
- [Olm Tusky Helm chart](../charts/olm-tusky/)

