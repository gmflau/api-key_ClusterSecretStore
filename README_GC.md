# api-key authorization with Google Cloud Secret Manager

Set up Google Cloud specific environment variables:
```sh
export PROJECT_ID=$(gcloud config get-value project)\export 
export SERVICE_ACCOUNT=521162177580-compute@developer.gserviceaccount.com

#export SERVICE_ACCOUNT=1092811355202-compute@developer.gserviceaccount.com
#export SERVICE_ACCOUNT=glau-solo@customer-success-386314.iam.gserviceaccount.com
```

Create a secret and store your API key
```sh
gcloud secrets create glau-api-key --replication-policy="automatic"
```

Add the secret version with the API key value::
```sh
echo -n "CheckItOut" | gcloud secrets versions add glau-api-key --data-file=-
```

Grant Access to the Service Account:
```sh
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:$SERVICE_ACCOUNT" \
    --role="roles/secretmanager.secretAccessor" \
    --condition 'None'
```

Install the External Secrets Operator:
```sh
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm install external-secrets external-secrets/external-secrets
```

Create a K8 secret that stores the Google Cloud service account creds:
```sh
kubectl create secret generic gcp-secret-manager-credentials -n httpbin --from-file=key.json=./third-carving-148900-23390a32cb6c.json
```

Create a ClusterSecretStore Resource for Google Cloud Secret Manager:
```sh
kubectl apply -f- <<EOF
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: gcp-secrets-store
spec:
  provider:
    gcpsm:
      projectID: $PROJECT_ID
      auth:
        secretRef:
          secretAccessKeySecretRef:
            name: gcp-secret-manager-credentials
            key: key.json
EOF
```

Create an ExternalSecret Resource:
```sh
kubectl apply -f- <<EOF
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-gcp-secret
  namespace: httpbin
spec:
# RefreshInterval is the amount of time before the values reading again from the SecretStore provider
  # Valid time units are "ns", "us" (or "µs"), "ms", "s", "m", "h" (from time.ParseDuration)
  # May be set to zero to fetch and create it once
  refreshInterval: "20s"
  secretStoreRef:
    name: gcp-secrets-store
    kind: ClusterSecretStore
  target:
    name: my-k8s-secret
    creationPolicy: Owner
    template:
      type: extauth.solo.io/apikey
      metadata:
        labels:
            team: infrastructure
  data:
    - secretKey: api-key
      remoteRef:
        key: glau-api-key
EOF
```

