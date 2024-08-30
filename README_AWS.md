# api-key authorization with AWS Secrets Manager

Set environment variables:
```sh
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export REGION=us-west-1
export EKS_CLUSTER=glau-dev-cluster3
export SECRET_NAME=glau-api-key
export IAM_USER=my-k8s-secrets-user
export EXTERNAL_SECRET_ACCESS_POLICY=external-sercert-access-policy
export ESO_SERVICE_ACCOUNT=eso-service-account
```

Create an EKS cluster:
```sh
eksctl create cluster --name $EKS_CLUSTER \                              
--spot --version=1.29 \
--region $REGION --nodes 2 --nodes-min 0 --nodes-max 3 \
--instance-types t3.large \        
--tags created-by=gilbert_lau,team=csa,purpose=customer-success 
```

Associates an IAM OIDC identity provider with your EKS cluster, which allows your cluster to authenticate using [IAM roles for service accounts (IRSA)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html):
```sh
eksctl utils associate-iam-oidc-provider --cluster $EKS_CLUSTER --approve
```

Create a Secret for an api-key in AWS Secrets Manager:
```sh
aws secretsmanager create-secret --name $SECRET_NAME --secret-string "{\"api-key\": \"LetMeIn\"}" --region $REGION
```
Verify the secret:
```sh
aws secretsmanager get-secret-value --secret-id $SECRET_NAME --region $REGION
```

Install the External Secrets Operator:
```sh
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm install external-secrets external-secrets/external-secrets
```

Create an IAM Policy json file for External Secrets Operator:
```sh
cat <<EOF > eso_policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret",
        "secretsmanager:ListSecretVersionIds",
		    "secretsmanager:GetResourcePolicy"
      ],
      "Resource": "*"
    }
  ]
}
EOF
```
Creat the IAM Policy:
```sh
aws iam create-policy \
  --policy-name $EXTERNAL_SECRET_ACCESS_POLICY \
  --policy-document file://eso_policy.json
```
Create a Kubernetes Service Account with the policy created above:
```sh
eksctl create iamserviceaccount \
  --name $ESO_SERVICE_ACCOUNT \
  --namespace default \
  --cluster $EKS_CLUSTER \
  --attach-policy-arn arn:aws:iam::$ACCOUNT_ID:policy/$EXTERNAL_SECRET_ACCESS_POLICY \
  --approve \
  --override-existing-serviceaccounts
```
Configure External Secrets Operator to use the IAM Role:
```sh
kubectl patch deployment external-secrets -n default -p "{\"spec\": {\"template\": {\"spec\": {\"serviceAccountName\": \"$ESO_SERVICE_ACCOUNT\"}}}}"
```

Create the ClusterRoleBinding so the service account should have the necessary permissions to list ExternalSecret resource:
```sh
kubectl apply -f- <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: externalsecrets-access-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-secrets-controller
subjects:
  - kind: ServiceAccount
    name: $ESO_SERVICE_ACCOUNT
    namespace: default
EOF
```
Verify the service account has `list` permission on ExternalSecret resource:
```sh
kubectl auth can-i list externalsecrets.external-secrets.io --as=system:serviceaccount:default:$ESO_SERVICE_ACCOUNT
```

Create Cluster Secret Store for AWS:
```sh
kubectl apply -f- <<EOF
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secret-store
spec:
  provider:
    aws:
      service: SecretsManager
      region: $REGION
      auth:
        jwt:
          serviceAccountRef:
            name: $ESO_SERVICE_ACCOUNT
            namespace: default
EOF
```
Create External Secret in Kubernetes:
```sh
kubectl apply -f- <<EOF
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-aws-ext-secret
  namespace: default
spec:
  refreshInterval: "1m"
  secretStoreRef:
    name: aws-secret-store
    kind: ClusterSecretStore
  target:
    name: my-aws-k8s-secret
    creationPolicy: Owner
    template:
      type: extauth.solo.io/apikey
      metadata:
        labels:
            team: infrastructure
  data:
    - secretKey: api-key
      remoteRef:
        key: $SECRET_NAME
        property: api-key
EOF
```


Tear down the EKS cluster:
```sh
eksctl delete cluster --name $EKS_CLUSTER
```