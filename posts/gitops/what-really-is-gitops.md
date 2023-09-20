---
title: What Really is GitOps?
description: Welcome   Welcome to the second post in our series on GitOps.  You'll find the first post -...
tags: 'beginners, cloudnative, azure, devops'
published: true
id: 1605139
date: '2023-09-19T15:59:54.606Z'
---
## Welcome

Welcome to the second post in our series on GitOps.  [You'll find the first post - Just Enough Git for GitOps - here](https://aka.ms/cloudnative/JustEnoughGit).

GitOps describes the process and tooling around managing the state of a Kubernetes cluster based on content in a Git repository.  We aren't going to get into specific tooling in this post - that'll start tomorrow with a post on FluxCD (with lots more to follow over the course of the next week and a half).

## GitOps is ...

### Defining GitOps

There are a number of definitions for GitOps (kinda like how we had a bajillion definitions for DevOps).

[Steve Buchanan](https://twitter.com/buchatech) (of [bucahtech.com] and a teammate here at Microsoft) defines GitOps as:

> GitOps is an operating model pattern for cloud native applications storing application & declarative infrastructure code in Git as the source of truth used for automated continuous delivery.

### The Principles of GitOps

The [OpenGitOps standards group](https://opengitops.dev/) defines GitOps using a set of four principles:

#### 1) Declarative

A system managed by GitOps must have it's state expressed declaratively.

#### 2) Versioned and Immutable

Desired state is stored in a way that enforces immutability, versioning and retains a complete version history.

#### 3) Pulled Automatically

Software agents automatically pull the desired state declarations from the source.

#### 4) Continuously Reconciled

Software agents continuously observe actual system state and attempt to apply the desired state.

### Elements of a GitOps Environment

#### Bare Minimum

GitOps, at a minimum, requires a Git repository and an agent (usually an operator) in your Kubernetes cluster.  The agent's role is to poll for configuration changes between the state of configuration files in a particular branch in the Git repo and the state of the cluster.

Example:

![Basic architecture for GitOps with Flux](https://learn.microsoft.com/en-us/azure/architecture/example-scenario/gitops-aks/media/gitops-flux.png)

#### More Realistic Environment

The basic example ignores the container build process and image deployment.  A more realistic example may look like:

![More complete architecture for GitOps with Flux](https://learn.microsoft.com/en-us/azure/architecture/example-scenario/gitops-aks/media/gitops-ci-cd-flux.png)

### The Essence of GitOps

The GitOps principles defined by the OpenGitOps group drills down to the core of what GitOps is:

A Git repository with a specified branch protected from change by technical measures and process.

An agent (operator) that monitors both the cluster configuration and the specified branch to detect change and remediate configuration differences.

## Why GitOps?

Why would we want to use GitOps? I'll call back to the [OpenGitOps group and what they enumerate as the potential benefits of using GitOps tools and practices](https://opengitops.dev/about).

* Increased Developer & Operational Productivity
* Enhanced Developer Experience
* Improved Stability
* Higher Reliability
* Consistency and Standardization
* Stronger Security Guarantees

### Is GitOps Different from DevOps?

GitOps seems to have a lot in common with traditional DevOps practices.  

If we go back to the [Continuous Delivery](https://www.continuousdelivery.com/) concept, we see one of the foundations is Configuration Management - which is where GitOps tooling and process fall.

GitOps is for Kubernetes what Chef/Puppet/PowerShell DSC/CF Engine/and others were for server configuration management.

So, GitOps isn't a replacement or different concept from DevOps - it's tooling and process specialized for working with Kubernetes environments.

## What I Like about GitOps

Coming from an infrastructure as code background, I really like the concept of the GitOps operator performing consistency checks and remediation of configuration drift over my infrastructure.

Using a source control driven process also makes me happy (though not completely... see below).  There are tremendous benefits (see all the State of DevOps reports) to operational stability and reductions in mean time to recovery when your infrastructure configuration is a version-able artifact.

## My Problem(s) with GitOps

I have one philosophical problem with GitOps and one operational problem.

### My Operational Problem with GitOps

Let's start with the operational problem, as that's more of an observation based on many years in the infrastructure as code space.

Typically, most teams I've worked with have a very difficult time giving control of environmental and operational changes to an agent. One of the reasons tools like Ansible or active deployment by CD tools were so popular was that it was a "push" process where some operator or developer action was required to initiate the process to deploy or change configuration.

When folks attempted to operate agent-based configuration management tools where the agent ran on some schedule and performed a "test and repair" process, the concern was often "what if someone made a manual change?".  While this isn't a technical problem with the tooling, it is a people and process problem.  Folks had a difficult time (either personally or organizationally) allowing the agent to take control and then having to trust (and monitor) the status of those agent operations.

Some of this boiled down to a lack of trust in the configuration management tooling (not the agent, but the instructions the agents had to follow).  This trust could be built by quality testing - which was often neglected.

Again, this isn't a technical problem with GitOps, but it is a potential blocker if teams cannot buy into funneling all changes through the agent and staying away from the `kubectl` command. 

### My Philosophical Problem with GitOps

One of the core principles of GitOps is that the configuration is "versioned and immutable" and this versioning provides an accurate audit log.

This pushes the responsibility of being an artifact repository to the Git repository, which is not the purpose of the tooling.  We have to rely on hosting solutions and process to enforce process to make sure the repository can achieve those goals.

Additionally, managing application configuration and secrets requires special handling and forces us to build process outside of the GitOps workflow to ensure auditability and versioning for those values.

Fortunately, there are options to move "GitOps" artifacts to actual artifact stores (like OCI artifacts), I just wish that was the default.

## Summary

GitOps is an umbrella for practices and tooling to use Git-based configuration elements to manage our Kubernetes environment via an agent that manages the cluster configuration.

Adopting GitOps practices and tools can improve your operational capabilities but requires some forethought and buy in from your team.

Join us tomorrow to [take a look at Flux](https://aka.ms/cloudnative/GitGoingWithGitOps) with [Paul Yu](https://dev.to/pauldotyu)!
