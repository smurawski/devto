---
title: Deep Dive into Working with Draft
description: What is Draft and what does it do?   Draft exists to help folks get up and running on...
tags: 'cloudnative, beginners, azure'
published: true
id: 1587073
date: '2023-09-01T17:51:24.501Z'
---
## What is Draft and what does it do?

[Draft](https://github.com/azure/draft) exists to help folks get up and running on Kubernetes more easily. 

This command line tool is run in your project's working directory and attempts to identify the type of project in the repository.

From there, Draft will ask you some questions and generate all the files you need to containerize the application and deploy into a Kubernetes cluster.

In addition to the Dockerfile, Draft can provide `Helm` charts, `kustomize` configuration files, or plain old Kubernetes manifests.

Once your project has those configuration files, Draft can help you set up GitHub actions to build, package, and deploy your application into a Kubernetes cluster.


## How does Draft Work?

Draft has several subcommands that we'll look at in depth.  

First is `draft create` which will generate the Dockerfile and any supporting configuration files necessary.  

Second is `draft setup-gh` which configures [GitHub OpenID Connect](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-azure) with Microsoft Entra ID (formerly Azure Active Directory). 

The third subcommand is `draft generate-workflow` which creates GitHub workflow files to build, package, and deploy your application into Kubernetes. 

The fourth main subcommand is `draft update` which can update your configuration files to allow your application to be accessible from outside the cluster.  

There are a few other minor features that we'll review as well.

### Setting up an Azure Environment

To create a basic environment to test with Draft, we can use the Bicep template that I've included in [my sample repo](https://github.com/smurawski/drafttesting).  It will create a resource group called `drafttesting` with an AKS cluster and Azure Container Registry in the region you specify.

```
git clone https://github.com/smurawski/drafttesting
cd drafttesting

# I deployed to eastus, but pick a region that fits for you.
az deployment sub create --template-file ./deploy/main.bicep --location eastus
```

### Setting up a test project

To start, I'm going to create a simple Go project. (or feel free to copy it from [my sample repo](https://github.com/smurawski/drafttesting)

```
mkdir DraftTesting
cd DraftTesting
go mod init DraftTesting
```

And add a file `main.go` (which I borrowed from [learn.microsoft.com](https://learn.microsoft.com/azure/app-service/quickstart-golang))

```
package main
import (
    "fmt"
    "net/http"
)
func main() {
    http.HandleFunc("/", HelloServer)
    http.ListenAndServe(":8080", nil)
}
func HelloServer(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, %s!", r.URL.Path[1:])
}
```

### Getting Started with `draft create`

After [installing Draft](https://github.com/Azure/draft/#installation), while still in the `DraftTesting` directory, I'll run `draft create`.  

#### Dockerfile generation

Draft detects that I've got a Go project and asks if I want to use Go Modules.  Other languages will have appropriate questions asked for their specific needs.

```
~/source/DraftTesting ❯  draft create
[Draft] --- Detecting Language ---
v yes
[Draft] --> Draft detected Go (88.475836%)
```

Once the language is identified, Draft moves on to create a Dockerfile.  You'll be asked for the port to expose in the Dockerfile.  My app is listening on 8080, so I'll use that.  Then it asks for the version of Go my module requires, which for me was 1.21.  

```
[Draft] --- Dockerfile Creation ---
Please enter the port exposed in the application (default: 80): 8080
Please enter the version of go used by the application (default: 1.18): 1.21.1
[Draft] --> Creating Dockerfile...
```
Now, I have a Dockerfile.

```
FROM golang:1.21
ENV PORT 8080
EXPOSE 8080

WORKDIR /go/src/app
COPY . .

RUN go mod vendor
RUN go build -v -o app
RUN mv ./app /go/bin/

CMD ["app"]
```
#### Generating Kubernetes Manifests

Draft continues to ask me which type of deployment I want to use.  I'll keep it simple and pick `manifests`.  

```
[Draft] --- Deployment File Creation ---
v manifests
```

In one of the more annoying bits of the workflow, I get asked for the exposed port again.  

```
Please enter the port exposed in the application: 8080
```

Draft prompts me to enter a name for the application.  

```
Please enter the name of the application: drafttesting
```

Continuing on with Draft's fascination with ports, it asks which port should be used to expose my application outside the cluster, though it makes the default my previously selected port.

```
Please enter the port the service uses to make the application accessible from outside the cluster (default: 8080):
```

Next, I'm asked for the namespace to deploy the application into.  I took the default.

```
Please enter  the namespace to place new resources in (default: default):
```

This leads to the first question where things could start to go REALLY wrong if we aren't paying attention. Draft wants to know the image name for the deployment.  Here's where we'll need to know what container registry our application is going to be deployed to. 

Thanks to the Bicep template we ran earlier, I do have an Azure Container Registry to which I can deploy my container.  We need to prefix our container image name with URL of the container registry.

```
Please enter the name of the image to use in the deployment (default: drafttesting): acrxms72d.azurecr.io/drafttesting
```

Finally, we provide the tag to deploy.

```
Please enter the tag of the image to use in the deployment (default: latest): v1
```

Now, we have our manifests to deploy our application into the Kubernetes cluster.

```
[Draft] --> Creating manifests Kubernetes resources...

[Draft] Draft has successfully created deployment resources for your project 
[Draft] Use 'draft setup-gh' to set up Github OIDC.
```

Draft created a `manifests` folder in my project directory with a `deployment.yaml` and a `service.yaml`.  I'm not going to reproduce them here, but if you want a bit more on these files, check out [Kubernetes Fundamentals - Pods and Deployments](https://azure.github.io/Cloud-Native/cnny-2023/fundamentals-day-1) and [Kubernetes Fundamentals - Services and Ingress](https://azure.github.io/Cloud-Native/cnny-2023/fundamentals-day-2).

#### So, what templates are used by `draft create`?

The templates are embedded in the tool and can be found in [the source on GitHub](https://github.com/Azure/draft).  The Dockerfile templates are [here](https://github.com/Azure/draft/tree/main/template/dockerfiles) and the deployment templates are [here](https://github.com/Azure/draft/tree/main/template/deployments).

### Securing Access to Azure for my GitHub Workflow with `draft setup-gh`

Draft suggested we move onto setting up GitHub OpenID Connect (OIDC), so let's get that done.

#### OpenID Connect with Microsoft Entra ID/Azure Active Directory

First, let's define what's going to happen. Draft is first going to create an Microsoft Entra App Registration.

Then, it will configure a federated credential for that app registration.  The federated credential will define what GitHub resources can request a token to access my subscription via the application.

Finally, it creates a service principal for that application and assigns it permissions to the resource group specified.  The application will use that service principal to access the resources for GitHub Actions.

#### Running `draft setup-gh`

First, Draft prompts me to enter an app registration name.

```
Enter app registration name: drafttesting
```

Then, I can pick the subscriptions I have access to based on my current `az` CLI login session.

```
Enter app registration name: drafttesting         
Use the arrow keys to navigate: ↓ ↑ → ←
v my-azure-subscription-guid
```

Then, I need to provide a resource group to provide access to.  My AKS cluster and ACR instance are in the `drafttesting` resource group I created earlier.

```
Enter resource group name: drafttesting
```

Next, we need to enter our GitHub organization (or user) and repository.  As you can see, my naming skills are on point and very clever.

```
Enter github organization and repo (organization/repoName): smurawski/drafttesting
```

After that, Draft setups up OIDC between Microsoft Entra ID and GitHub.  This can take a minute or two.

```
[Draft] Draft has successfully set up Github OIDC for your project 😃
[Draft] Use 'draft generate-workflow' to generate a Github workflow to build and deploy an application on AKS.
```

#### Troubleshooting the Federated Credential

To see what federated credential was created and the values supplied by Draft, you can use the Azure CLI.

```
applicationName=drafttesting
objectId=$(az ad app list --display-name $applicationName --query '[].id' -o tsv)
az ad app federated-credential list --id $objectId
```

This will output a configuration for your federated credential. Take note of the subject property.  That's where the configuration for what can access the credential lives.  You'll see values like `repo:smurawski/drafttesting:ref:refs/heads/main` which means that for requests originating from changes to my `main` branch, Microsoft Entra ID can provide access to my subscription.

The most likely mismatch will be attempting to deploy from a branch that was not specified when using `draft setup-gh`. 

### Generating Build Automation with `draft generate-workflow`

With our GitHub OIDC set up and configured to allow access to my Azure subscription, we can create our workflow.

Draft first asks what type of deployment manifests we are using. We selected manifests earlier, so we'll keep that choice now.

```
DraftTesting on  main ❯  draft generate-workflow
[Draft] --> Generating Github workflow
v manifests
```

Next, we are prompted for our Azure Container Registry name, our container image name, the resource group for our AKS cluster, and the cluster name.

```
Please enter the Azure container registry name: acrxms72d
Please enter the container image name: drafttesting
Please enter the Azure resource group of your AKS cluster: drafttesting
Please enter the AKS cluster name: aks1           
```

Then we are prompted to enter which branch to deploy from and where in the project repository should be our Docker build context (I specified the default which is the project folder root).

```
Please enter the Github branch to automatically deploy from: main
Please enter the path to the Docker build context (default: .):
```

Draft then generated our workflow file and ensured that our deployment manifest had the correct repository and deployment, but it removed the tag.  This is not a problem as the build created will replace the image at deployment time with one tagged and built in that GitHub Actions run.

```
[Draft] Draft has successfully generated a Github workflow for your project 😃
```

#### So, what templates are used by `draft generate-workflow`?

The templates used for the workflows can be found [here](https://github.com/Azure/draft/tree/main/template/workflows)

### Allowing Traffic to my Application with `draft update`

The `draft update` subcommand supports enabling extensions to AKS.  Currently, only [`webapp_routing`](https://learn.microsoft.com/azure/aks/app-routing?tabs=without-osm) is supported.

You'll need a TLS certificate stored in Azure Key Vault to use this feature.

You can find the templates for this feature [here](https://github.com/Azure/draft/tree/main/template/addons/azure/webapp_routing).

### Other `draft` subcommands

There are a few other subcommands available. 

`draft completion` will provide autocompletion helpers for your particular shell.

```
DraftTesting on  main ❯  draft completion --help
Generate the autocompletion script for draft for the specified shell.
See each sub-command's help for details on how to use the generated script.


Usage:

  draft completion [command]


Available Commands:

  bash         Generate the autocompletion script for bash
  fish         Generate the autocompletion script for fish
  powershell   Generate the autocompletion script for powershell
  zsh          Generate the autocompletion script for zsh
```

`draft info` will dump out all the support answers for the languages/runtimes and deployment options.

```
DraftTesting on  main ❯  draft info
{
  "supportedLanguages": [
    {
      "name": "javascript",
      "displayName": "JavaScript",
      "variableExampleValues": {
        "VERSION": [
          "10.16.3",
          "12.16.3",
          "14.15.4"
        ]
      }
    },
    {
      "name": "erlang",
      "displayName": "Erlang",
      "variableExampleValues": {
        "BUILDERVERSION": [
          "24.2-alpine"
        ],
        "VERSION": [
          "3.15"
        ]
      }
    },
    {
      "name": "go",
      "displayName": "Go",
      "variableExampleValues": {
        "VERSION": [
          "1.16",
          "1.17",
          "1.18",
          "1.19"
        ]
      }
    },
    {
      "name": "php",
      "displayName": "PHP",
      "variableExampleValues": {
        "BUILDERVERSION": [
          "1"
        ],
        "VERSION": [
          "7.1-apache"
        ]
      }
    },
    {
      "name": "gradlew",
      "displayName": "Gradle",
      "variableExampleValues": {
        "BUILDERVERSION": [
          "jdk8",
          "jdk11",
          "jdk17",
          "jdk19"
        ],
        "VERSION": [
          "8-jre",
          "11-jre",
          "17-jre",
          "19-jre"
        ]
      }
    },
    {
      "name": "java",
      "displayName": "Java",
      "variableExampleValues": {
        "BUILDERVERSION": [
          "3-jdk-11"
        ],
        "VERSION": [
          "8-jre",
          "11-jre",
          "17-jre",
          "19-jre"
        ]
      }
    },
    {
      "name": "python",
      "displayName": "Python",
      "variableExampleValues": {
        "ENTRYPOINT": [
          "app.py",
          "main.py"
        ],
        "VERSION": [
          "3.9",
          "3.8",
          "3.7",
          "3.6"
        ]
      }
    },
    {
      "name": "ruby",
      "displayName": "Ruby",
      "variableExampleValues": {
        "VERSION": [
          "3.1.2",
          "2.6",
          "2.5",
          "2.4"
        ]
      }
    },
    {
      "name": "rust",
      "displayName": "Rust",
      "variableExampleValues": {
        "VERSION": [
          "1.70.0",
          "1.65.0",
          "1.60",
          "1.54",
          "1.53"
        ]
      }
    },
    {
      "name": "clojure",
      "displayName": "Clojure",
      "variableExampleValues": {
        "VERSION": [
          "8-jdk-alpine",
          "11-jdk-alpine",
          "17-jdk-alpine",
          "19-jdk-alpine"
        ]
      }
    },
    {
      "name": "gradle",
      "displayName": "Gradle",
      "variableExampleValues": {
        "BUILDERVERSION": [
          "jdk8",
          "jdk11",
          "jdk17",
          "jdk19"
        ],
        "VERSION": [
          "8-jre",
          "11-jre",
          "17-jre",
          "19-jre"
        ]
      }
    },
    {
      "name": "swift",
      "displayName": "Swift",
      "variableExampleValues": {
        "VERSION": [
          "5.2",
          "5.5"
        ]
      }
    },
    {
      "name": "csharp",
      "displayName": "C#",
      "variableExampleValues": {
        "VERSION": [
          "3.1",
          "4.0",
          "5.0",
          "6.0"
        ]
      }
    },
    {
      "name": "gomodule",
      "displayName": "Go Module",
      "variableExampleValues": {
        "VERSION": [
          "1.16",
          "1.17",
          "1.18",
          "1.19"
        ]
      }
    }
  ],
  "supportedDeploymentTypes": [
    "manifests",
    "helm",
    "kustomize"
  ]
}
```

And, of course there's `draft version` which will dump out the version information for the build you are using.

```
DraftTesting on  main ❯  draft version
version:  0.0.33
runtime SHA:  2142dbc745ac8284999f3384358680c7bc6f8570
```

## Try it out!

Head over to [the project releases](https://github.com/Azure/draft/releases) and grab the version for your system.  Try it out and give us some feedback via Issues or drop me a comment here!
