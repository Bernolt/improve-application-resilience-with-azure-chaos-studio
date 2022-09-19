# Improve application resilience with Azure Chaos Studio

Let's deploy a simple Azure Kubernetes Services cluster and a simple app to run a chaos experiment against, so you can familiarize yourself with Azure Chaos Studio.
But Before you can start deploying, there are some prerequisites to fulfill:
* You'll need the Azure command-line interface for deploying the resources, and you can find more information about the Azure CLI in the [documentation](https://docs.microsoft.com/en-us/cli/azure/).
* You'll need the Kubernetes command-line tool for deploying images and run other commands against your Kubernetes cluster. You can find more information about kubectl in the [documentation](https://kubernetes.io/docs/reference/kubectl/kubectl/).
* You'll need the Helm command-line tool for the deployment and configuration of applications and services. You can find more information about helm in the [documentation](https://helm.sh/docs/intro/install/).

This repository is part of the [Improve application resilience with Azure Chaos Studio](#) blog post.

## Create the Azure Kubernetes Cluster

**Step 1:** Download the ARM templates from the supporting GitHub repository for the rest of this guide. Review and adjust the templates and commands when needed. We won't be covering the details in this post.

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

*Step 5:* Set your environment variables and create a resource group. 

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

*Step 6:* Set the folder you've downloaded the ARM templates as your working directory and run the following code to deploy the cluster. Wait for the deployment to finish. This may take a couple of minutes.

```
az deployment group create \
  --name $DEPLOYMENT_NAME \
  --resource-group $RESOURCE_GROUP_NAME \
  --template-file "./aks-cluster.json" \
  --parameters location=$LOCATION resourceName=$AKS_NAME dnsPrefix=$DNS_PREFIX agentCount=1 agentVMSize=$NODE_SIZE
```

*Step 7:* Review what is deployed. You'll find a resource group that contains the service, node pools, and networking. And that you have a separate resource group that contains the nodes, a load balancer, public IP addresses, network security group (NSG), managed identity, virtual network, and route table.

## Deploy the web application 

**Step 1:** Set the environment variables using your terminal from the previous steps.

```
$ACR_NAME="acrtaksweuchaosrobino01"
$DEPLOYMENT_NAME="AcrDeploymentRobino-01"
```

**Step 2:** Run the following code to deploy the container registry using the ARM template. Wait for the deployment to finish. This may take a couple of minutes.

```
az deployment group create \
  --name $DEPLOYMENT_NAME \
  --resource-group $RESOURCE_GROUP_NAME \
  --template-file "./acr.json" \
  --parameters registryLocation=$LOCATION registryName=$ACR_NAME
```

**Step 3:** Once the container registry is deployed, it's time to import the container image of the sample application. Use the commands below to log in to your container registry and import the image.

```
az acr login -n $ACR_NAME --expose-token
```

```
az acr import --name $ACR_NAME \
  --source mcr.microsoft.com/azuredocs/azure-vote-front:v2 \
  --image azure-vote-front:v2
```

Run the command below to validate that the container image has successfully been imported.

```
az acr repository list --name $ACR_NAME --output table
```

**Step 4:** We need to give our AKS cluster access to pull the image from our container registry. We'll use the managed identity of the cluster. Use the command below to get your managed identity's id representing your cluster's agent pool. From the returned values, copy its principal ID.

```
az identity list \
  --query "[?contains(name,'$AKS_NAME-agentpool')].{Name:name, PrincipalId:principalId}" \
  --output table
```

Use the command below to give the managed identity the AcrPull role on the resource group level. This allows it to pull images from the container registry, which we previously deployed within the same resource group as the AKS cluster. Ensure you've adjusted the principal id before executing the command.

```
az role assignment create \
  --assignee "<principalId>" \
  --role "AcrPull" \
  --resource-group $RESOURCE_GROUP_NAME
```

**Step 5:** The manifest file you've downloaded from the GitHub repository uses the image from the example container registry, which most likely won't exist when you follow these instructions. Open the `azure-vote-app.yml` manifest file with a text editor or IDE and run the following command to get your ACR login server name:

```
az acr list --resource-group $RESOURCE_GROUP_NAME --query "[].{acrLoginServer:loginServer}" --output table
```

Replace `acrtaksweuchaosrobino01.azurecr.io` with your ACR login server name. The image name is found on line 53 of the manifest file. Provide your own ACR login server name, ensuring your manifest file looks like the following example:

```
containers:
- name: azure-vote-front
  image: <acrName>.azurecr.io/azure-vote-front:v2
```

**Step 6:** Next, we'll deploy the application using the kubectl apply command. This command parses the manifest file and creates the defined Kubernetes objects. To do so, we need to get the access credentials for our AKS cluster by running the `az aks get-credentials` command. Run the commands below—having the folder which holds your manifest file as the working directory.  

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

Once completed, you should have two pods, `azure-vote-front` and `azure-vote-back`, in a running state. You can validate this by running the command below.

```
kubectl get pods
```

**Step 7:** You should now have a working voting app published to the internet over HTTP. Run the command below.

```
kubectl get service
```

Copy the `EXTERNAL_IP` of the `azure-vote-front` service, and paste it into your browser. Congratulations, you've now successfully deployed the Azure Voting App.﻿

## Set up Chaos Mesh on your AKS cluster

**Step 1:** Our first step will be adding the `chaos-mesh` chart repository and ensuring we have the latest update. Run the commands below to add the repository.

```
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm repo update
```

**Step 2:** To scope our Chaos Mesh pods, services, and deployments, we'll create a separate namespace. Run the command below to create the namespace.

```
kubectl create ns chaos-testing
```

**Step 3:** Next, we'll install Chaos Mesh into the namespace. Run the command below to install chaos mesh to your AKS cluster.

```
helm install chaos-mesh chaos-mesh/chaos-mesh --namespace=chaos-testing --set chaosDaemon.runtime=containerd --set chaosDaemon.socketPath=/run/containerd/containerd.sock
```

The installation will create multiple pods and services. Validate that the pods are running using the command below.

```
kubectl get pods -n chaos-testing
```

If everything went as intended, you've successfully installed Chaos Mesh on your AKS cluster. 

## Onboard the AKS cluster to Azure Chaos Studio

Run the command below to onboard the AKS cluster to Azure Chaos Studio. Ensure you have the folder that holds the template as your working directory.

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

To validate the successful onboarding, navigate to [Azure Chaos Studio Targets in the Azure Portal](https://portal.azure.com/#view/Microsoft_Azure_Chaos/ChaosStudioMenuBlade/~/targetsManagement), and see if your AKS cluster has the service-direct option enabled.

## Create an Azure Chaos Studio experiment

**Step 1**: Open the `experiment.json` file in your IDE or text editor and inspect it. We will adjust anything. However, it's good to understand how the deployment works, especially the parts on lines 61 and 62, which contains the `jsonSpec`. This contains a JSON-escaped Chaos Mesh spec that uses the [PodChaos kind](https://chaos-mesh.org/docs/simulate-pod-chaos-on-kubernetes/#create-experiments-using-yaml-configuration-files).

**Step 2:** Add the environment variables, using your terminal from the previous steps. You can use the commands below.

```
$EXPERIMENT_NAME="exp-t-aks-weu-chaos-pod-robino-01"
$DEPLOYMENT_NAME="ExperimentDeploymentRobino-01"
```

**Step 3:** Run the following code to deploy the chaos experiment. The deployment should complete in a matter of seconds. 

```
az deployment group create \
  --name $DEPLOYMENT_NAME \
  --resource-group $RESOURCE_GROUP_NAME \
  --template-file "./experiment.json" \
  --parameters resourceName=$EXPERIMENT_NAME location=$LOCATION clusterName=$AKS_NAME
```

You have now successfully deployed your chaos experiment. If you like, you can [review your chaos experiment in the Azure Portal](https://portal.azure.com/#view/Microsoft_Azure_Chaos/ChaosStudioMenuBlade/~/chaosExperiment) before going to the next step.

**Step 4:** To run the experiment, it needs the permissions for the target resource. More information about the required permissions can be found in the [documentation](https://docs.microsoft.com/en-us/azure/chaos-studio/chaos-studio-fault-providers).

Run the command below to retrieve the experiment's id, so we can use it for assigning the required permissions in the next step.

```
az ad sp list \
  --display-name $EXPERIMENT_NAME \
  --query [].id --output table
```

**Step 5:** Run the command below to give the experiment the `Azure Kubernetes Service Cluster Admin Role` on the resource group. Make sure you've adjusted the principal id before executing the command.

```
az role assignment create \
  --assignee "<principalId>" \
  --role "Azure Kubernetes Service Cluster Admin Role" \
  --resource-group $RESOURCE_GROUP_NAME
```

## You're done!
We're finally ready to run the freshly created chaos experiment.
