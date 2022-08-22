# Improve application resilience with Azure Chaos Studio

Let's deploy a simple Azure Kubernetes Services cluster and a simple app to run a chaos experiment against, so you can familiarize yourself with Azure Chaos Studio.

This repository is part of the [Improve application resilience with Azure Chaos Studio](#) blog post.

## Create the Azure Kubernetes Cluster

**Step 1:** Download the ARM-templates for the rest of this guide from this GitHub repository. Review and adjust the templates to your needs if you like. 

**Step 2:** Start your preferred terminal and connect to Azure using the Azure CLI and your identity (user account).

```
az login
```

**Step 3:** List the available subscriptions and locate the preferred subscription id.

```
az account list --output table
```

**Step 4:** Set the subscription you're using to deploy the resources. Use the subscription id from the previous step.

```
az account set --subscription <subscriptionID>
```

*Step 5:* Set your environment variables and create a resource group. You can use the commands below and adjust them to your liking.

```$RESOURCE_GROUP_NAME="rg-t-aks-weu-chaos-robino-01"
$AKS_NAME="aks-t-weu-chaos-robino-01"
$DNS_PREFIX="aks-t-weu-chaos-robino-01-dns"
$LOCATION="westeurope"
$NODE_SIZE="Standard_B2s"
$DEPLOYMENT_NAME="AksDeploymentRobino-01"
```

```
az group create \
  --name $RESOURCE_GROUP_NAME \
  --location "$LOCATION"
```

*Step 6:* Set the folder you've downloaded the ARM templates as your working directory and run the following code to deploy the cluster. Wait for the deployment to finish, this may take a couple of minutes.

```
az deployment group create \
  --name $DEPLOYMENT_NAME \
  --resource-group $RESOURCE_GROUP_NAME \
  --template-file "./aks-cluster.json" \
  --parameters location=$LOCATION resourceName=$AKS_NAME dnsPrefix=$DNS_PREFIX agentCount=1 agentVMSize=$NODE_SIZE
```

## Deploy a web application 

**Step 1:** Set the environment variables, using your terminal from the previous steps. You can use the commands below and adjust them to your liking.

```
$ACR_NAME="acrtaksweuchaosrobino01"
$DEPLOYMENT_NAME="AcrDeploymentRobino-01"
```

**Step 2:** Run the following code to deploy the container registry, using the ARM-template you've downloaded. Wait for the deployment to finish, this may take a couple of minutes.

```
az deployment group create \
  --name $DEPLOYMENT_NAME \
  --resource-group $RESOURCE_GROUP_NAME \
  --template-file "./acr.json" \
  --parameters registryLocation=$LOCATION registryName=$ACR_NAME
```

**Step 3:** Once the deployment of the container registry is done, it is time to import a container image of the sample application. Use the commands below to login on your container registry and import the container image.

```
az acr login -n $ACR_NAME --expose-token
```

```
az acr import --name $ACR_NAME \
  --source mcr.microsoft.com/azuredocs/azure-vote-front:v2 \
  --image azure-vote-front:v2
```

To validate that the container image has successfully been imported, run the command below.

```
az acr repository list --name $ACR_NAME --output table
```

**Step 4:** We need to give our AKS cluster access to pull the image from our container registry. We'll use the managed identity of the AKS cluster to do so. Use the command blow to get the ID of your managed identity that represents the agentpool in your AKS cluster. From the returned values, copy its principal ID.

```
az identity list \
  --query "[?contains(name,'$AKS_NAME-agentpool')].{Name:name, PrincipalId:principalId}" \
  --output table
```

Use the command below to give the managed identity the AcrPull role on the resource group level. This allows it to pull images from our container registry, which we previously deployed within the same resource group as the AKS cluster. Make sure you adjust the principal ID before executing the command.

```
az role assignment create \
  --assignee "<principalId>" \
  --role "AcrPull" \
  --resource-group $RESOURCE_GROUP_NAME
```

**Step 5:** The manifest file you've downloaded from this GitHub repository uses the image from my example container registry, which most likely won't exist at the time you are following these instructions. Open the manifest file (azure-vote-app.yml) with a text editor or IDE you like. Run the following command to get your ACR login server name:

```
az acr list --resource-group $RESOURCE_GROUP_NAME --query "[].{acrLoginServer:loginServer}" --output table
```

Replace *acrtaksweuchaosrobino01.azurecr.io* with your ACR login server name. The image name is found on line 53 of the manifest file. Provide your own ACR login server name, so that your manifest file looks like the following example:

```
containers:
- name: azure-vote-front
  image: <acrName>.azurecr.io/azure-vote-front:v2
```

**Step 6:** Next up, we'll deploy the application using the kubectl apply command. This command parses the manifest file and creates the defined Kubernetes objects. In order to do so, we need to get the access credentials for our AKS cluster, by running the az aks get-credentials command. Run the commands below, having the folder which holds your manifest file as the working directory. 

```
az aks get-credentials -g $RESOURCE_GROUP_NAME -n $AKS_NAME
```

```
kubectl apply -f azure-vote-app.yml --validate=false
```

This may take a few minutes to complete. Use the command below to monitor the deployment status.

```
kubectl rollout status deployment/azure-vote-front
```

Once completed, you should have two pods (azure-vote-front and azure-vote-back) in a running state. You can validate this by running the command below.

```
kubectl get pods
```

**Step 7:** You should now have a working voting app, which is published to the internet over HTTP. Run the command below.

```
kubectl get service
```

Copy the EXTERNAL_IP of the azure-vote-front service, and paste it in your browser. Congratulations, you've now successfully deployed your Azure Voting App.

## Set up Chaos Mesh on your AKS cluster

**Step 1:** Our first step will be adding a new chart repository for chaos-mesh, and making sure we have the latest update. Run the commands below to add the repository.

```
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm repo update
```

**Step 2:** To create a new scope for our chaos mesh pods, services, and deployments in the cluster, we'll create a separate namespace for chaos mesh. Run the command below to create the namespace.

```
kubectl create ns chaos-testing
```

**Step 3:** Next, we'll install chaos mesh into the freshly created namespace. Don't mind the flags in the command below. We won't be covering the details in this post, but you won't have to adjust them. Run the command below to install chaos mesh to your AKS cluster.

```
helm install chaos-mesh chaos-mesh/chaos-mesh --namespace=chaos-testing --set chaosDaemon.runtime=containerd --set chaosDaemon.socketPath=/run/containerd/containerd.sock
```

The installation will create multiple pods and services. Before continuing, validate that all pods are running, using the command below.

```
kubectl get pods -n chaos-testing
```

If everything went as intended, you have now successfully installed chaos mesh to your AKS cluster. 

## Onboard the AKS cluster to Azure Chaos Studio

Run the command below to onboard the AKS cluster to Azure Chaos Studio, while making sure you have the folder that holds the template as your working directory.

```
$DEPLOYMENT_NAME="OnboardDeploymentRobino-01"
```

```
az deployment group create \
  --name $DEPLOYMENT_NAME \
  --resource-group $RESOURCE_GROUP_NAME \
  --template-file "./onboard.json" \
  --parameters resourceName=$AKS_NAME resourceGroup=$RESOURCE_GROUP_NAME
```

To validate that the onboarding was successful, navigate to [Azure Chaos Studio Targets in the Azure Portal](https://portal.azure.com/#view/Microsoft_Azure_Chaos/ChaosStudioMenuBlade/~/targetsManagement), and see if your AKS cluster has service-direct option enabled. 


