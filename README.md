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
