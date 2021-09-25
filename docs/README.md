# Secure OpenShift Deployment using Tekton Pipelines

Tekton is an open source project to configure and run CI/CD pipelines within a OpenShift/Kubernetes cluster.

## Overview

| Module      | Description                          |
| ----------- | ------------------------------------ |
| 0. Setup | [setup](lab1/0_setup.md) |
| 1. Create a Basic Pipeline using Tekton | [lab1](lab1/1_clone-git-repo.md) |
| 2. Add SAST and SCA | lab2 (coming soon) |
| 3. Build, Sign and Push Images to a Private Repository | lab3 (coming soon)  |
| 4. Scan Images for Vulnerabilities | lab4 (coming soon)  |
| 5. Scan Running Containers in OpenShift | lab5 (coming soon)  |
| 6. Governance, Risk and Compliance (GRC) using Security and Compliance Center (SCC) | lab6 (coming soon)  |

## Introduction

In this learning path you learn:

* the basic concepts in Tekton pipelines,
* how to create a pipeline to build an image and deploy a container,
* how to run the pipeline, check its status and troubleshoot problems,
* how to `Shift Left` security and include security checks into your pipeline,
* add Static Application Secuirty Testing (SAST),
* add Software Component Analysis (SCA),
* push images to a private repository,
* sign images,
* scan images for vulnerabilities,
* scan running containers for vulnerabilities,
* check compliance of your deployments against NIST 850-3 and SOC2 regulations,

Also, check out this [very good tutorial](https://github.com/openshift/pipelines-tutorial) by Red Hat.

## Technologies

This tutorial was tested using:

* IBM Cloud Shell, image version 1.0.28
* IBM Cloud CLI, version 2.0.3
* Container Registry plugin, version 0.1.543
* OpenShift, version 4.6
* OpenShift CLI, client version 4.6.23
* Tekton, client version 0.12.0
  * task/git-clone/0.3
  * task/kaniko/0.2

## Estimated time

1 hour

## About Tekton

[Tekton](https://tekton.dev) is an open-source framework for creating CI/CD pipelines. Tekton is governed by the [Continuous Delivery Foundation (CDF)](https://cd.foundation), which includes among other the following special interest groups:

* Best practices,
* Interoperability,
* MLOps,
* Security.

Tekton provides a set of extensions of the Kubernetes API for defining pipelines, in the form of [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/). Tekton consists of about 15 projects like Pipelines, Triggers, Results, Chains, Operator, Catalog, Dashboard, CLI and Hub.

The following diagram shows the basic resources in Tekton.

![crd](images/crd.png)

The resources are used as follows.

* A **PipelineRun** defines an execution of a pipeline. It references the **Pipeline** to run.
* A **Pipeline** defines the set of **Tasks** that compose a pipeline.
* A **Task** defines a set of build steps such as compiling code, running tests, and building and deploying images.

A Task executes a Pod and is a sequence of Steps. A Step is equivalent to a Container. You can define the name, image, environment and a script to run inside the container.  All Steps in a Task can access a shared workspace, an implicit volume of the pod.

A Pipeline is a collection of Tasks that can run sequentially or conditionally. Pipeline has access to a shared persistent volume (workspace) and combines tasks through the workspace and results to certain output. The pipeline also provides a finalizer function.

Pipeline and Tasks are executed by PipelineRun and TaskRun. To automatically invoke these you can create a Trigger. A TriggerBinding extracts data from the event payload, and a TriggerTemplate creates the template for the PipelineRun. The EventListener is a Custom Resource Definition (CRD) that combines the TriggerBinding and the TriggerTemplate. You can find existing pipeline resources in the Tekton Catalog or Tekton Hub.

### Tekton at IBM

* IBM Cloud Continuous Delivery,
* IBM Code Engine,
* Watson AI Pipelines,
* IBM Datastage ETL services,
* RedHat OpenShift Pipelines.

## Contributors

* Oliver Rodriguez, [github](https://github.com/odrodrig),
* Remko de Knikker, [github](https://github.com/remkohdev),
* Steve Martinelli, [github](https://github.com/stevemar),
* John Zaccone, [github](https://github.com/jzaccone),
* David Carew, [github](https://github.com/djccarew),
* Greg Dritschler, [github](https://github.com/GregDritschler),
* Olaph Wagoner, [github](https://github.com/loafyloaf).
