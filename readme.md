# Microsoft Hands-on

# Create AKS cluster

```bash
export AZURE_APP_ID=aaaa
export AZURE_TENANT_ID=bbbb
export AZURE_PASSWORD=ccccc
export AZURE_SUBSCRIPTION=your-subscription-id

export CLUSTER_NAME=microsoft

# Login to azure
az login --service-principal --username ${AZURE_APP_ID} --password ${AZURE_PASSWORD} --tenant ${AZURE_TENANT_ID}

# Create grup in azure for the hands on
az group create --name ${CLUSTER_NAME} --location westeurope

# Create aks cluster
az aks create --resource-group ${CLUSTER_NAME} --name ${CLUSTER_NAME} --node-count 3 --ssh-key-value=~/.ssh/id_rsa --subscription ${AZURE_SUBSCRIPTION} --service-principal ${AZURE_APP_ID} --client-secret ${AZURE_PASSWORD}

# Retrieve aks credentials
az aks get-credentials --resource-group ${CLUSTER_NAME} --name ${CLUSTER_NAME} --file ~/.kube/${CLUSTER_NAME}.yaml

# Create ad app with reply URLs
az ad app create --display-name ${CLUSTER_NAME} --reply-urls https://dashboard.microsoft.demo.traefiklabs.tech/callback https://app.microsoft.demo.traefiklabs.tech/callback

# Retrive the appId
AD_APP_ID=$(az ad app list --display-name ${CLUSTER_NAME} --query '[0].appId' | tr -d '"')

# Generate new credentials
az ad app credential reset --id ${AD_APP_ID}
{
  "appId": "xxxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxx",
  "name": "xxxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxx",
  "password": "random_password",
  "tenant": "yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy"
}

# The appId and password need to be add in the configmap `gitops/01-configmpa.yaml`
```


## TraefikEE

### Gitops installation

```bash
# Create traefikee namespace
kubectl create namespace traefikee

# Create secret for the license
kubectl create secret -n traefikee generic microsoft-license --from-literal=license="${TRAEFIKEE_LICENSE}"

# Retrieve the yaml file for the installation
curl -so gitops/00-enterprise.yaml "https://install.enterprise.traefik.io/v2.4?cluster=microsoft&namespace=traefikee&staticconfig=static.toml"

# Adapt static configuration in the configmap
vi gitops/01-configmap.yaml

# Apply the gitops file
k apply -f gitops/

# Annotate proxies pods to enable prometheus scrapping.
for item in $(kubectl -n traefikee get pods -l component=proxies --output=name); do
kubectl annotate -n traefikee ${item} prometheus.io/scrape='true';
kubectl annotate -n traefikee ${item} prometheus.io/port='8080';
done

# Generate credential to connect teectl to the cluster
kubectl exec -n traefikee microsoft-controller-0 -- /traefikee generate credentials --kubernetes.kubeconfig="${KUBECONFIG}"  --cluster=microsoft > config.yaml

# Import generated credentials
teectl cluster import --file="config.yaml" --force

# Use the imported cluster
teectl cluster use --name microsoft
```

## Demo

### Exposing the dashboard

Deploy the dashboard and show it to the user

```
k apply -f traefikee/ingress.yaml
```

### App v1

deploy the demo application and show how to expose this application with an IngressRoute.

```bash
k apply -f app/v1/
```

After showing that the application is exposed, explain how to enable middleware on this IngressRoute to add headers, add distributed middlewares and add security with OIDC, JWT or LDAP.
Adapte the file [app/v1/02-ingress.yaml](app/v1/02-ingress.yaml) to match the customer needs.

#### Authentication Middlewares

After configuring Keycloak, use the following scripts to demonstrate JWT and OAuth 2.0 middlewares:

##### JWT

```bash
# set environment variables
KEYCLOAK_USR=""
KEYCLOAK_PWD=""
CLIENT_SECRET=""

# get JWT from Keycloak server
JWT=$(curl -L -X POST "https://keycloak.microsoft.demo.traefiklabs.tech/auth/realms/traefiklabs/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=demo-app" \
  -d "client_assertion_type=" \
  -d "grant_type=password" \
  -d "client_secret=$CLIENT_SECRET" \
  -d "scope=openid" \
  -d "username=$KEYCLOAK_USR" \
  -d "password=$KEYCLOAK_PWD" | jq -r '.access_token')

# curl app with JWT in Authorization Header
curl -H "Authorization: Bearer $JWT" \
  https://app.microsoft.demo.traefiklabs.tech
```

##### OAuth 2.0

Add the BASIC_AUTH value to the oAuthSource authorizationHeader.

```bash
BASIC_AUTH=$(echo -n "demo-app:$CLIENT_SECRET" | base64)

curl -L -X POST \
  -H "Authorization: Basic $BASIC_AUTH" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "token_type_hint=access_token&token=$JWT" \
  https://keycloak.microsoft.demo.traefiklabs.tech/auth/realms/traefiklabs/protocol/openid-connect/token/introspect | jq

# curl app with JWT in Authorization Header
curl -H "Authorization: Bearer $JWT" \
  https://app.microsoft.demo.traefiklabs.tech
```

### Open API

In this second scenario, explain how to use the API portal to reference the services's OpenAPI specification.

```bash
k apply -f app/openapi
k apply -f app/openapi/old
```

## Clean

Don't forget to stop your environment.

```bash
# Delete AKS cluster
az aks delete --name ${CLUSTER_NAME} --resource-group ${CLUSTER_NAME} --yes

# Delete group
az group delete --name ${CLUSTER_NAME} --yes

# Delete the ad app
az ad app delete --id ${AD_APP_ID}
```
