# Azure Functions KEDA Walkthrough

## Overview

The walkthrough is designed to take a user step by step through the process for containerizing and deploying an Azure Function to AKS, using KEDA for autoscaling purposes. The process includes a few steps:

- Containerize a local Azure Function
- Deploy KEDA to the AKS cluster
- Create the Kubernetes manifest file
- Apply the manifest to the AKS cluster
- View the Function working in AKS
- Debug KEDA

## Pre-Requisites
- A working, local Azure Function [(Docs)](https://docs.microsoft.com/en-us/azure/azure-functions/functions-create-first-function-vs-code?pivots=programming-language-csharp)
- An empty AKS cluster [(Docs)](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough)

## Containerize a local Azure Function
To deploy the Azure Function to AKS, an image must be built for it first. In order to create the image, a Dockerfile will need to be created for the project. While the Dockerfile can be built manually, the Azure Functions Core Tools give users the ability to automatically create a Dockerfile for a given Azure Function project through the ```func init --docker-only``` command. ```func init``` is a command commonly run to initialize a new Azure Functions project from scratch. Because we already possess a working Azure Function, we do not need to initialize the project, and only need to create the Dockerfile. ```func init --docker-only``` skips all of the initialization steps for the Azure Function, instead only creating a Dockerfile and .dockerignore file. The Core Tools will determine the language and version of the Azure Function and will create a Dockerfile specific to those values. 

In order to create the Dockerfile for your project, run ```func init --docker-only``` from inside your Azure Functions project.

**IMPORTANT**
As of April 1, 2020 Azure Functions Core Tools only automatically create the Dockerfile for v2 functions that are written in .NET, Node, Python, or PowerShell. In order to create Dockerfile for Java, or for v3 functions, the Dockerfile must be written manually. Use the Azure Functions [image list](https://github.com/Azure/azure-functions-docker) to find the correct image to base the Azure Function off of. 

## Deploy KEDA to the AKS cluster

First, connect to the empty cluster:

```az aks get-credentials --resource-group <resource-group-name> --name <cluster-name>```

The best way to deploy KEDA to an AKS cluster is via Helm. Helm and alternative deployment instructions can be found on the [KEDA deployment docs](https://keda.sh/docs/deploy/#helm).

_Note: Ensure that the 'keda' namespace is created and that the KEDA resources deployed via Helm are done so into this namespace._

## Create the Kubernetes manifest file

The Azure Function can be quickly deployed to the AKS cluster in a single step by using the command ```func kubernetes deploy --name <name-of-function-deployment> --registry <container-registry-username>```. While this _can_ be accomplished in a single step, in order to fully understand the process, the walkthrough will instead take a step by step approach. 

Let's break down what this command is doing:
1. The Dockerfile created earlier is used to build a local image on your machine
2. The local image is tagged and pushed to a container registry where the user is logged in. **container-registry-username** is the user's username on that registry (e.g. Docker Hub)
3. A manifest is created and applied to the cluster

First, create a local image for the Azure Function by running the command ```docker build -t <container-registry-username>/my-function-image .``` from inside the Azure Functions project. Ensure that the image is tagged appropriately in preparation for a push to the remote registry.

Next, the image will need to be pushed to the remote registry. Confirm that you have logged into your registry of choice before advancing. Push the image to the remote registry using the command ```docker push <container-registry-username>/my-function-image```.

The ```func kubernetes deploy``` command essentially accepts 2 arguments - the ***name*** of the deployment, which can be set to any value, and a second reference to a remote image. The first example used the `--registry <container-registry-username>` flag to reference the remote image, because there were intermediary steps that pushed the image to the registry behind the scenes. Instead, if the image already exists on a remote registry, a direct reference can be made with the ```--image-name <remote-image>``` flag. The function _could_ now be immediately deployed to the AKS cluster using the command ```func kubernetes deploy --name <name-of-function-deployment> --image-name <remote-image>```, but do not do so yet.

To reiterate, once the Core Tools has access to a remote image, it will create and apply a Kubernetes manifest file to the AKS cluster. Enterprise grade Azure Function AKS deployments will most likely need to incorporate the manifest file into source control, and may require additional manipulation of the file. The ```--dry-run``` flag forces the Core Tools to output the manifest file to the console, rather than automatically applying it to the cluster. 

To output the manifest to a file for further processing, run ```func kubernetes deploy --name <name-of-function-deployment> --image-name <remote-image> --dry-run > deploy.yml```

## Apply the manifest to the AKS cluster

At this point, the image exists on a remote registry and a deploy.yml file has been created inside the Azure Functions project, containing the Kubernetes manifest that describes the resources to be applied to the AKS cluster. Examining the contents of the file, there should be 3 resources:
1. SECRET - A series of base64 encoded values, taken from the local.settings.json file, are deployed to the cluster via a Secret resource.
2. DEPLOYMENT - A Deployment resource is created based on the remote image that was pushed to the registry. It has access to environment variables that match the key value pairs in the Secret resource, using the envFrom property.
3. SCALED OBJECT - The ScaledObject resource connects the Deployment resource to KEDA's autoscaling capabilities. The Azure Functions Core Tools automatically determines the trigger type for the Azure Function and creates an appropriate [Scaler](https://keda.sh/#scalers). The ScaledObject resource contains trigger property values derived from the Azure Function, some of which depend on the Deployment's environment variables.

In order to apply the manifest to the cluster, simply run ```kubectl apply -f deploy.yml```

## View the Function working in AKS

Initially, the Deployment will scale to 0 pods, and the Azure Function will appear as though it is not running. To view the running pods, run ```kubectl get pods```. Unless events were already being sent to the trigger source, the output should look  like:

![No pods running](./images/NoPods.jpg)

Send events to the trigger source to prompt KEDA to scale up the Functions deployment, and run ```kubectl get pods -w``` to view the autoscaling in real time. The output should soon look like:

![No pods running](./images/AutoscalingPods.jpg)

_Note: The Deployment will scale back down to 0 running pods once the events have been processed._

## Debug KEDA

In the event that KEDA does not autoscale the Azure Functions Deployment, the KEDA operator pod should be debugged. To find the KEDA pods running in the cluster, run ```kubectl get pods -A```. The KEDA related resources were deployed into the 'keda' namespace and the ```-A``` flag lists _all_ pods, rather than those in the default namespace. The output should look like:

![No pods running](./images/ViewKEDAPods.jpg)

KEDA deploys 2 pods into the cluster - the operator and operator-metrics pods. To view the KEDA logs, in order to debug an autoscaling issue, the logs should be viewed for the _operator_ pod. In this case, the command would be ```kubectl -n keda logs keda-operator-6bdf8cbb68-mb6nc```


