---
title: Getting Started with Hippo - a WebAssembly PaaS (Part 1)
description: 'Setting up a Development Environment   Hey y’all, I’ve recently been dipping into the cloud...'
tags: 'rust, webassembly, beginners'
published: true
id: 901199
date: '2021-11-17T22:39:56.714Z'
---

# Setting up a Development Environment

Hey y’all, I’ve recently been dipping into the cloud native space and wanted to share one of the projects I’m looking into. In this series, we’ll set up a development environment, explore how we can interact with that environment, and then port a little CLI tool I built to run as an application in a WebAssembly PaaS called Hippo.

One of the interesting developments has been the focus on WebAssembly (or WASM) as a secure and portable engine. You may have heard about WebAssembly as a way to improve the performance of web applications that run in the browser, which it can do and does well. WASM is also being looked at to run workloads in other contexts. Right out of the box, WASM brings a lightweight engine and a very strong sandbox model. There’s a standard being defined for how that sandbox can interact with the system around it, the WebAssembly System Interface (WASI). This standard paved the way for an implementation, called Wasmtime, which is now being used in a number of different projects, one of which we’ll dig into here.

## What is ...?

Before we get too far into the weeds, there are lot of acronyms and projects, we’ve seen two already in the first paragraph and more are coming. Let’s define a few things here, so if you’re like me and take a time or two to get them straight, there’s a place to go for reference.

WASM or WebAssembly, a compact, low-level language with a binary format, initially designed to run in the browser. There are a multitude of ways to get a WebAssembly file (a .wasm file for a binary output or .wat file with a textual representation). There are, according to [webassmbly.org](https://webassembly.org/getting-started/developers-guide/), twelve or thirteen programming languages that support creating WebAssembly outputs. We’ll look at using Rust in this exploration.

WASI or WebAssembly System Interface, which is a standard for how the WebAssembly engine can be exposed to the underlying system resources. It offers a controlled path to system resources and requires explicit delegation of resources to the WebAssembly engine. It covers capabilities like file system access, networking, and other core operating system features (like random number generation).

[Wasmtime](https://github.com/bytecodealliance/wasmtime) is an implementation of a WASM runtime (hence wasmtime) that implements the WASI standard. The project provides the C API defined by the WASI standard, as well as providing a place to experiment with new features which may end up in the standard. Wasmtime can be used in projects in a variety of languages, offering bindings for Rust, Python, Go, and .NET as part of the project and Java, Perl, Zig, and Ruby as external projects. There are two toolchains for targeting Wasmtime, one in C/C++ and one in Rust.

[WAGI](https://github.com/deislabs/wagi) or the WebAssembly Gateway Interface. Since the WASI standard doesn’t have network support defined yet, WAGI takes the [Common Gateway Interface standard](https://datatracker.ietf.org/doc/html/rfc3875) and provides the infrastructure to route network requests to WASM modules configured as HTTP handlers.

[Bindle](https://github.com/deislabs/bindle) is an aggregate object store – allowing you to create versioned artifacts and store them with any and all the files needed. Bindle provides the object store for Hippo.

[Hippo](https://github.com/deislabs/hippo) is a Platform-as-a-Service (PaaS) designed to provide a hosting environment that makes following cloud-native best practices easier. Hippo uses WAGI to provide a sandboxed, secure, and highly performant runtime environment with a great developer experience.

Enough with the definitions and background, let’s get started building an application for WASM and hosting it on Hippo.

## Environment Setup

Fortunately, my colleague [Scott Coulton](https://github.com/scotty-c) has made this easier for us. We can go to his [Hippo-Dev](https://github.com/scotty-c/hippo-dev) repository where we have two ways to get started. First, for a local development experience, we could use [Multipass](https://multipass.run/) and

```bash
curl -L -o cloud-init.yaml "https://raw.githubusercontent.com/scotty-c/hippo-dev/blob/main/cloud-init.yaml"

multipass launch --name hippo-server --cpus 2 --mem 4G --disk 20G --cloud-init cloud-init.yaml
```

Our second option, and the route I’ll take, is to spin up a VM in Azure. We’ll need [an Azure subscription](https://azure.microsoft.com/free?WT.mc_id=containers-44762-stmuraws) and the [Az CLI](https://docs.microsoft.com/cli/azure?WT.mc_id=containers-44762-stmuraws). Then,

```bash
curl -L -o deploy-azure.sh "https://raw.githubusercontent.com/scotty-c/hippo-dev/main/deploy-azure.sh"

bash deploy-azure.sh
```

I had to create a new resource group to start out.

![Terminal with the Az CLI command to create a new resource group.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hrmslhinn4i1t012yk8m.png)

Then the script will prompt you for your subscription ID, the resource group you want to use, and the name of the VM you want to deploy.

![Terminal showing the deployment script with prompts for needed information](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ve3x009gct8bf0x733ji.png)

In either case, the cloud-init script will handle setting up a development environment on that server, along with Hippo and Bindle servers running. After the deploy happens (and we wait a few moments for the build of Hippo to complete), we should be able to access the Hippo web portal on port 5001.

![Terminal with completed deployment results](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/dvmlr8tbxepg9iglzxur.png)

Let’s create a new user from the registration link on the login page.

![Hippo dashboard login screen](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pjn5oojui26ybkmz242r.png)

I’ll use the same login credentials as the Hippo tutorial (admin as the user and Passw0rd! for the password), as we’ll start there as we get our first deployment up and running, but that’s our next post!

![Hippo dashboard new user registration](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/q0mdvgju57rlm9j499x8.png)

We now have a server with Hippo running and a full suite of development tools to explore this new runtime environment.
