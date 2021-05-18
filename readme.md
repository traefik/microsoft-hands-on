# Microsoft Hands-on

# Create AKS cluster

```bash
export AZURE_APP_ID=aaaa
export AZURE_TENANT_ID=bbbb
export AZURE_PASSWORD=ccccc
export AZURE_SUBSCRIPTION=your-subscription-id
export AZURE_LOCATION=westeurope

export CLUSTER_NAME=gbb
export PREFIX=${CLUSTER_NAME}-$(openssl rand -hex 12 | tr '[:upper:]' '[:lower:]')
export DOMAIN=${PREFIX}.${AZURE_LOCATION}.cloudapp.azure.com
export LE_EMAIL=michael@traefik.io

# Login to azure
az login --service-principal --username ${AZURE_APP_ID} --password ${AZURE_PASSWORD} --tenant ${AZURE_TENANT_ID}

# Create group in azure for the hands on
az group create --name ${CLUSTER_NAME} --location ${AZURE_LOCATION}

# Create aks cluster
az aks create --resource-group ${CLUSTER_NAME} --name ${CLUSTER_NAME} --node-count 3 --ssh-key-value=~/.ssh/id_rsa.pub --subscription ${AZURE_SUBSCRIPTION} --service-principal ${AZURE_APP_ID} --client-secret ${AZURE_PASSWORD}

# Retrieve aks credentials
az aks get-credentials --resource-group ${CLUSTER_NAME} --name ${CLUSTER_NAME} --file ~/.kube/${CLUSTER_NAME}.yaml

# Create ad app with reply URLs
az ad app create --display-name ${CLUSTER_NAME} --reply-urls "https://${DOMAIN}/callback"

# Retrieve the appId
AD_APP_ID=$(az ad app list --display-name ${CLUSTER_NAME} --query '[0].appId' | tr -d '"')

# Add required Microsoft Graph permissions
az ad app permission add \
  --api 00000003-0000-0000-c000-000000000000 \
  --api-permissions e1fe6dd8-ba31-4d61-89e7-88639da4683d=Scope 37f7f235-527c-4136-accd-4a02d197296e=Scope 64a6cdd6-aab1-4aaf-94b8-3cc8405e90d0=Scope 14dad69e-099b-42c9-810b-d002981feec1=Scope \
  --id ${AD_APP_ID}

# Generate new credentials
az ad app credential reset --id ${AD_APP_ID}
# Sample response - appId, password, and tenant values need to be added to the configmap `gitops/01-configmap.yaml`
# {
#   "appId": "xxxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxx",
#   "name": "xxxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxx",
#   "password": "random_password",
#   "tenant": "yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy"
# }
```

# TraefikEE

Prerequisites:
- AKS cluster up and ready
- Custom DNS name
- teectl [installed](https://doc.traefik.io/traefik-enterprise/installing/teectl-cli/)

## Gitops installation

```bash
# Create traefikee namespace
kubectl create namespace traefikee

# Create secret for the license
kubectl create secret -n traefikee generic ${CLUSTER_NAME}-license --from-literal=license="${TRAEFIKEE_LICENSE}"

# Retrieve the yaml file for the installation
curl -so gitops/00-enterprise.yaml "https://install.enterprise.traefik.io/v2.4?cluster=${CLUSTER_NAME}&namespace=traefikee&staticconfig=static.toml"

# Adapt static configuration in the configmap
vi gitops/01-configmap.yaml
sed -i "s/LE_EMAIL/${LE_EMAIL}/g" gitops/01-configmap.yaml
sed -i "s/CLUSTER_NAME/${CLUSTER_NAME}/g" gitops/01-configmap.yaml

# Apply the gitops file
kubectl apply -f gitops/

# Add annotation to the LoadBalancer service to create DNS entry
kubectl annotate service ${CLUSTER_NAME}-proxy-svc -n traefikee service.beta.kubernetes.io/azure-dns-label-name=${PREFIX}

# Check DNS entry creation
dig ${DOMAIN} # Shoudl return an A entry on the LB IP

# Generate credential to connect teectl to the cluster
kubectl exec -n traefikee ${CLUSTER_NAME}-controller-0 -c ${CLUSTER_NAME}-controller -- /traefikee generate credentials --kubernetes.kubeconfig="${KUBECONFIG}"  --cluster=${CLUSTER_NAME} > config.yaml

# Import generated credentials
teectl cluster import --file="config.yaml" --force

# Use the imported cluster
teectl cluster use --name ${CLUSTER_NAME}
```

## Demo

### Exposing the dashboard

Deploy the dashboard.

```bash
sed -i "s/DOMAIN/${DOMAIN}/g" traefikee/ingress.yaml

kubectl apply -f traefikee/ingress.yaml
```

### Demo App

Deploy the demo application and expose this application with an IngressRoute.

```bash
sed -i "s/DOMAIN/${DOMAIN}/g" app/02-ingress.yaml

kubectl apply -f app/
```

With the application deployed, we can now add the following items:
- Middlewares:
  - Custom header
  - Rate limiting
  - OpenID Connect authentication

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
