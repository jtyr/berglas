# Berglas Cloud Run Example - Ruby

This guide assumes you have followed the [setup instructions][setup] in the
README. Specifically, it is assumed that you have created a project, Cloud
Storage bucket, and Cloud KMS key.

[setup]: https://github.com/GoogleCloudPlatform/berglas#setup

1. Make sure you are in the `examples/cloudrun/ruby` folder before
continuing!

1. Export the environment variables for your configuration:

    Using Secret Manager storage:

    ```text
    export PROJECT_ID=my-project
    ```

    Using Cloud Storage storage:

    ```text
    export PROJECT_ID=my-project
    export BUCKET_ID=my-bucket
    export KMS_KEY=projects/${PROJECT_ID}/locations/global/keyRings/berglas/cryptoKeys/berglas-key
    ```

1. Create two secrets using the `berglas` CLI (see README for installation
instructions):

    Using Secret Manager storage:

    ```text
    berglas create sm://${PROJECT_ID}/api-key "xxx-yyy-zzz"
    ```

    ```text
    berglas create sm://${PROJECT_ID}/tls-key "=== BEGIN RSA PRIVATE KEY..."
    ```

    Using Cloud Storage storage:

    ```text
    berglas create ${BUCKET_ID}/api-key "xxx-yyy-zzz" \
      --key ${KMS_KEY}
    ```

    ```text
    berglas create ${BUCKET_ID}/tls-key "=== BEGIN RSA PRIVATE KEY..." \
      --key ${KMS_KEY}
    ```

1. Create a dedicated service account for the Cloud Run service:

    ```text
    gcloud iam service-accounts create "cloudrun-berglas-ruby" \
      --project ${PROJECT_ID}
    ```

    ```text
    export SA_EMAIL=cloudrun-berglas-ruby@${PROJECT_ID}.iam.gserviceaccount.com
    ```

1. Grant the service account access to the secrets:

    Using Secret Manager storage:

    ```text
    berglas grant sm://${PROJECT_ID}/api-key --member serviceAccount:${SA_EMAIL}
    berglas grant sm://${PROJECT_ID}/tls-key --member serviceAccount:${SA_EMAIL}
    ```

    Using Google Cloud storage:

    ```text
    berglas grant ${BUCKET_ID}/api-key --member serviceAccount:${SA_EMAIL}
    berglas grant ${BUCKET_ID}/tls-key --member serviceAccount:${SA_EMAIL}
    ```

1. Build a container using Cloud Build and publish it to Container Registry:

    ```text
    gcloud builds submit \
      --project ${PROJECT_ID} \
      --tag gcr.io/${PROJECT_ID}/berglas-example-ruby:0.0.1 \
      .
    ```

1. Deploy the container on Cloud Run:

    ```text
    gcloud run deploy berglas-example-ruby \
      --project ${PROJECT_ID} \
      --platform managed \
      --region us-central1 \
      --image gcr.io/${PROJECT_ID}/berglas-example-ruby:0.0.1 \
      --memory 1G \
      --concurrency 10 \
      --set-env-vars "API_KEY=berglas://${BUCKET_ID}/api-key,TLS_KEY=berglas://${BUCKET_ID}/tls-key?destination=tempfile" \
      --service-account ${SA_EMAIL} \
      --allow-unauthenticated
    ```

1. Access the service:

    ```text
    curl $(gcloud run services describe berglas-example-ruby
      --project ${PROJECT_ID} \
      --platform managed \
      --region us-central1 \
      --format 'value(status.address.url)')
    ```

1. (Optional) Cleanup the deployment:

    ```text
    gcloud run services delete berglas-example-ruby \
      --quiet \
      --project ${PROJECT_ID} \
      --platform managed \
      --region us-central1
    ```

    ```text
    IMAGE=gcr.io/${PROJECT_ID}/berglas-example-ruby
    for DIGEST in $(gcloud container images list-tags ${IMAGE} --format='get(digest)'); do
      gcloud container images delete --quiet --force-delete-tags "${IMAGE}@${DIGEST}"
    done
    ```

1. (Optional) Revoke access to the secrets:

    Using Secret Manager storage:

    ```text
    berglas revoke sm://${PROJECT_ID}/api-key --member serviceAccount:${SA_EMAIL}
    berglas revoke sm://${PROJECT_ID}/tls-key --member serviceAccount:${SA_EMAIL}
    ```

    Using Cloud Storage storage:

    ```text
    berglas revoke ${BUCKET_ID}/api-key --member serviceAccount:${SA_EMAIL}
    berglas revoke ${BUCKET_ID}/tls-key --member serviceAccount:${SA_EMAIL}
    ```

1. (Optional) Delete the service account:

    ```text
    gcloud iam service-accounts delete "${SA_EMAIL}" \
      --quiet \
      --project ${PROJECT_ID}
    ```
