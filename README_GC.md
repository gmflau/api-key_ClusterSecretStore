# api-key authorization with Google Cloud Secret Manager

Set up Google Cloud specific environment variables:
```sh
export PROJECT_ID=$(gcloud config get-value project)
export SERVICE_ACCOUNT=1092811355202-compute@developer.gserviceaccount.com
export SA_KEY=customer-success-386314-4b74df4f328b.json
export API_KEY=glau-api-key
```

Follow [Get started guide](https://docs.solo.io/gateway/latest/quickstart/) to install Gloo Gateway Enterprise Edition, set up an API gateway with a HTTP listener, and deploy the `httpbin` sample app.

Create a secret and store your API key:
```sh
gcloud secrets create $API_KEY --replication-policy="automatic"
```
Add the secret version with the API key value::
```sh
echo -n "CheckItOut" | gcloud secrets versions add $API_KEY --data-file=-
```

Grant Access to the Service Account to be used by the External Secrets Operator:
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
kubectl create secret generic gcp-secret-manager-credentials -n httpbin --from-file=key.json=./$SA_KEY
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
        key: $API_KEY
EOF
```
Verify the creation of a K8 secret:
```sh
kubectl get secret my-k8s-secret -n httpbin -oyaml
```
    
Verify the api-key authentication:
Create an Authconfig:
```sh
kubectl apply -f- <<EOF
apiVersion: enterprise.gloo.solo.io/v1
kind: AuthConfig
metadata:
  name: apikey-auth
  namespace: httpbin
spec:
  configs:
    - apiKeyAuth:
        headerName: api-key
        labelSelector:
          team: infrastructure
EOF
```
Create a RouteOption resource and reference the AuthConfig resource:
```sh
kubectl apply -f- <<EOF
apiVersion: gateway.solo.io/v1
kind: RouteOption
metadata:
  name: apikey-auth
  namespace: httpbin
spec:
  options:
    extauth:
      configRef:
        name: apikey-auth
        namespace: httpbin
EOF
```
Create an HTTPRoute for the httpbin that requires api-key authenticaiton:
```sh
kubectl apply -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: httpbin-apikey-auth
  namespace: httpbin
spec:
  parentRefs:
  - name: http
    namespace: gloo-system
  hostnames:
    - extauth.example
  rules:
    - filters:
        - type: ExtensionRef
          extensionRef:
            group: gateway.solo.io
            kind: RouteOption
            name: apikey-auth
      backendRefs:
        - name: httpbin
          port: 8000
EOF
```
```sh
export INGRESS_GW_ADDRESS=$(kubectl get svc -n gloo-system gloo-proxy-http -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $INGRESS_GW_ADDRESS
```
The following curl command will fail w/o providing the api-key and receive `401 Unauthorized` response:
```sh
curl -v http://$INGRESS_GW_ADDRESS:8080/status/200 -H "host: extauth.example:8080"
```
Run another curl command with the api-key with `200 OK` response:
```sh
curl -v http://$INGRESS_GW_ADDRESS:8080/status/200 -H "host: extauth.example:8080" \
-H "api-key: CheckItOut"
```