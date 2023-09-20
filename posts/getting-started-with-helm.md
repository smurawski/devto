---
title: Getting Started with Helm
description: 'Over the past few days, my team and I have looked into tools that can help you deploy containerized...'
tags: 'cloudnative, beginners, opensource, azure'
published: true
id: 1593880
date: '2023-09-08T17:52:31.130Z'
---
Over the past few days, my team and I have looked into tools that can help you deploy containerized workloads into Kubernetes more easily.

Here's a quick roundup of those posts:

- Video: [Deploying to Azure Container Apps](https://aka.ms/cloudnative/DeployingToACA)
- Lab: [Deploying to Azure Container Apps](https://aka.ms/cloudnative/DeployingToACALab)
- Blog post: [Jump Start Your Applications with Draft](https://aka.ms/cloudnative/BuildingWithDraft)
- Blog post: [Effortlessly Deploy to AKS with Open Source Tools Draft and Acorn](https://aka.ms/cloudnative/DraftAndAcorn)
- Lab: [Deploying Apps to AKS](https://aka.ms/cloudnative/DeployingToAKSLab)
- Blog post: [Deep Dive into Using Draft](https://aka.ms/cloudnative/draftdeepdive)
- Blog post: [Building Multi-Architecture Container Images](https://aka.ms/cloudnative/BuildingMultiArchContainers)
- Blog post: [Level-up Container Security: 4 Open-Source Tools for Secure Software Supply Chain](https://aka.ms/cloudnative/SecureSupplyChainTools)
- Blog post: [Pushing Multi-Architecture Container Images](https://aka.ms/cloudnative/PushingMultiArchContainers)

## Understanding Helm

Today, we are going to look at [Helm]. Helm can seem like a mystical tool, taking a bunch of marked up files, putting together a bunch of data about your environment, and then making a service appear in your cluster.

### What is Helm?

Helm is a package manager for Kubernetes. Helm provides a way to create templates for Kubernetes configuration files, apply those configurations with local data specific to your cluster, and manage updates, rollbacks, and other release concerns.  We'll focus on the packaging and deployment aspects of Helm in this article.

## Using Helm

Getting started with Helm is pretty straight forward. [Installing a local copy of Helm](https://helm.sh/docs/intro/install/) is the first step. 

Helm will also require you to have [`kubectl` installed](https://kubernetes.io/docs/tasks/tools/) and configured to access a Kubernetes cluster.

### Finding Helm Charts

The templates that Helm uses are called Charts.  Helm allows you to find and install charts from repositories.  Artifact Hub can help you find repositories ([Artifact Hub](https://artifacthub.io/packages/search?kind=0)). 

Step one is to add the chart repository.  We'll use my fork of the  [AKS Store Demo](https://github.com/smurawski/aks-store-demo) as an example.  We'll use the `helm repo add` command and provide it a (local) name for the repository, as well as the URL to the published chart repository.

```
aks-store-demo ❯ helm repo add aks-demo https://raw.githubusercontent.com/smurawski/aks-store-demo/helm/packages/
"aks-demo" has been added to your repositories
```

We can inspect what charts are available from the repository with `helm search repo`

```
aks-store-demo ❯ helm search repo aks-demo      
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
aks-demo/aks-store-demo 0.1.0           1.16.0          A Helm chart for Kubernetes
```

### Installing a Helm Chart

The simplest way to install a Helm chart, with no customization, is   with `helm install` with a name for the installation (called a release) and the specific chart to deploy.

```
aks-store-demo ❯ helm install aks-store-demo aks-demo/aks-store-demo
NAME: aks-store-demo
LAST DEPLOYED: Fri Sep  8 11:59:57 2023
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing aks-store-demo.

Your release is named aks-store-demo.

To learn more about the release, try:

  $ helm status aks-store-demo
  $ helm get all aks-store-demo
```

We can list the Helm releases with `helm list`.  These are namespace specific.

```
aks-store-demo ❯ helm list
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
aks-store-demo  default         1               2023-09-08 11:59:57.0813261 -0500 CDT   deployed        aks-store-demo-0.1.0    1.16.0
```

We can remove the release with `helm uninstall` followed by the name of the release.

```
aks-store-demo ❯ helm uninstall aks-store-demo 
release "aks-store-demo" uninstalled
```

#### Customizing the Installation

Helm charts typically have Values, which are default settings, that can be customized.

We can find the various options by using `helm show values` and providing the chart name.

```
aks-store-demo ❯ helm show values aks-demo/aks-store-demo
[LOTS OF YAML follows]
```

We can redirect this to a file and customize any of the values there.  Then, when we install the helm chart, we can point to our custom `values.yaml`.

```
aks-store-demo ❯ helm show values aks-demo/aks-store-demo > local_values.yaml
aks-store-demo ❯ # make a few changes in local_values.yaml (maybe change some replica counts)
aks-store-demo ❯ helm install aks-store-demo aks-demo/aks-store-demo -f local_values.yaml
... release details ...
```

That's the quick and dirty guide to getting up and running with Helm charts.  Now, we'll look into what it takes to convert our Kubernetes manifests into a Helm charts.

## Creating a Helm Chart

Helm offers a "quick" way to get started creating your own Helm charts with `helm create`.

```
mkdir charts
cd charts
helm create aks-store-demo
```

That's going to create a sample Chart with in the `aks-store-demo` directory.

In the directory will be an (empty) charts directory, a directory of templates (delete everything inside except for the `Notes.txt` file), a  `.helmignore` file, a `Chart.yaml` file, and a `values.yaml` file.  You can delete the contents of the `values.yaml` file.

In our template directory, we can drop any and all of our manifests.  In my example, I broke up the [all-in-one.yaml](https://github.com/smurawski/aks-store-demo/blob/helm/aks-store-all-in-one.yaml) into [individual manifests](https://github.com/smurawski/aks-store-demo/tree/helm/charts/aks-store-demo/templates).  To me, it was easier to experiment with the templating engine with smaller files.

### Converting a Manifest into a Template

Any of the manifests in the `templates` directory will be evaulated and applied as part of the `helm install`.  If a file does not have any templating features used, it'll just be applied as it exists.  For example, the `config_map_rabbitmq.yaml` doesn't have any values I want to change, so it just exists as it did in the `aks-store-all-in-one.yaml`.

The first thing I wanted to be able to customize is what container image to pull for each of the deployments.  I'll walk through how I changed one of the deployments, and you can explore the repo and files to see the changes made for the other deployments.

Let's use the `store-front` as our example.  I started out by identifying the default container image specified, which was `ghcr.io/azure-samples/aks-store-demo/store-front:latest`.

First, I opened my `values.yaml`, which should be empty (since we deleted all the contents earlier). I then added the following to the file and saved it.

```
store_front:
  image: 
    repository: ghcr.io/azure-samples/aks-store-demo/store-front
    tag: latest
```

The `values.yaml` is just a YAML file with whatever structure of data you feel will map effectively into your configs.

Then, I modified the `image` element of the deployment manifest to look like:

```
image: "{{ .Values.store_front.image.repository }}:{{ .Values.store_front.image.tag }}"
```

Next, I wanted to control the number of replicas for my deployment.  I updated `values.yaml` to look like:

```
store_front:
  replicaCount: 1
  image: 
    repository: ghcr.io/azure-samples/aks-store-demo/store-front
    tag: latest
```

Then, I updated the deployment manifest with:

```
replicas: {{ .Values.store_front.replicaCount }}
```

Since everything for this deployment is namespaced under `store_front` in my `values.yaml`, I can simplify this a bit by using `with`, which allows me to set a context from the `values.yaml` to use when referencing elements to apply to the template.

I added `{{ with .Values.store_front  }}` to the top of the manifest and `{{ end }}` to the bottom.  Then my two existing references need to be updated to look like this:

```
image: "{{ .image.repository }}:{{ .image.tag }}"
```

```
replicas: {{ .replicaCount }}
```

Now, that looks much simpler. :)

The last element I wanted to be able to customize is the resource requests and limits.  I moved those elements to the `values.yaml` like so:

```
store_admin:
  replicaCount: 1
  image: 
    repository: ghcr.io/azure-samples/aks-store-demo/store-admin
    tag: latest
  resources:
    requests:
      cpu: 1m
      memory: 200Mi
    limits:
      cpu: 1000m
      memory: 512Mi
```

And then I got to use my `with` again, along with the `toYaml` function to basically copy the YAML keys and values from the `values.yaml` into my template.  You'll notice there's a new element in the template, the `-`.  This tells the templating engine to only render that section if there's content for it from the `values.yaml`.  If there's no resources defined there, it'll leave that section out.

```
        {{- with .resources }}
        resources:
          {{- toYaml . | nindent 10 }}
        {{- end }}
```

### Testing Template Rendering

Once we've made a few updates to our templates, we can test to see how they come out.

From the root of the repository, you can run 

```
helm install aks-store-demo ./charts/aks-store-demo --dry-run --debug 
```

Which will evaluate and render the templates to the console, applying any values from the `values.yaml` file.

Another way to make sure you have valid templates is to use `helm lint`.  You'll have to be in the directory with the chart when the command executes. 

```
pushd ./charts/aks-store-demo
helm lint
popd
```

### The Notes.txt

I haven't mentioned it until now, but the `Notes.txt` gets rendered by the templating engine and written to the console upon the installation of a release of the chart.  You can include any information that folks may find useful in using what was installed by your chart.

## Closing

I hope this quick intro to Helm helps.  Please drop me a comment with any questions or feedback!

You'll find [my updates to the demo repository here](https://github.com/smurawski/aks-store-demo/tree/helm)
