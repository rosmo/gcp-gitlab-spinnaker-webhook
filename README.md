# gcp-gitlab-spinnaker-webhook

Google Cloud Function for proxying requests to
[OIDC-authenticated](https://openid.net/connect/) endpoints and in this case 
from Gitlab (or compatible systems) to Spinnaker protected by a GCP
[Identity Aware Proxy (IAP)](https://cloud.google.com/iap/) using a service
account.

Based on RealKinetic's [gcp-oidc-proxy](https://github.com/RealKinetic/gcp-oidc-proxy).

## Deployment

```sh
$ gcloud functions deploy gcp-gitlab-spinnaker-webhook \
    --runtime python37 \
    --entry-point handle_request \
    --trigger-http \
    --service-account SA@PROJECT_ID.iam.gserviceaccount.com \
    --set-env-vars=CLIENT_ID=...,TARGET_HOST=...,SECRET_KEY=...,WHITELIST=...
```

- The service account for the Cloud Function needs the "Service Account Token Creator" role.

  `gcloud iam service-accounts add-iam-policy-binding SA@PROJECT_ID.iam.gserviceaccount.com --member=serviceAccount:SA@PROJECT_IDiam.gserviceaccount.com --role=roles/iam.serviceAccountTokenCreator`

- A `CLIENT_ID` environment variable needs to be set containing the OAuth2
  client ID, e.g. the client ID used by IAP.
- A `WHITELIST` environment variable needs to be set containing a
  comma-separated list of paths to allow requests for. A value of `*` will
  whitelist all paths. Wildcards are supported.
- `TARGET_HOST` specifies the target host (eg. your Spinnaker hostname).
- `SECRET_HEADER` specifies the HTTP header where the webhook secret is taken from.
- `SECRET_KEY` specifies a JSON key where to put the secret token from the secret header.
- The service account for the Cloud Function needs to be added as a member of
  the protected resource with appropriate roles configured.
- Optionally, Basic authentication can be enabled by setting `AUTH_USERNAME`
  and `AUTH_PASSWORD` environment variables. If either of these is not set,
  authentication is disabled.

## Local Development

You can run the function locally with Functions Framework:

```sh
export TARGET_HOST=spinnaker.your.domain.com
export WHITELIST="/gate/webhooks/webhook/gitlab-trigger"
export CLIENT_ID=NUMBER-RANDOM.apps.googleusercontent.com
export SECRET_KEY="gitlab_token"
$ functions-framework --target=handle_request
```

You can submit test proxying locally by issuing a `curl` command:
```sh
$ curl -X POST -H 'Content-Type: application/json' \
    -d '{"X-Gitlab-Token":"..."}' -i \ 
    'http://username:password@localhost:8080/gate/webhooks/webhook/gitlab-trigger'
```

This will start an HTTP server which maps requests to the Cloud Function. This
requires setting the `GOOGLE_APPLICATION_CREDENTIALS` environment variable to a
service account credentials file which has the IAM roles described above.
