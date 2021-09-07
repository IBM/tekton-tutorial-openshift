# Setup

## Prerequisites

* Access to IBM Cloud shell or an equivalent client terminal,
* An instance of IBM Cloud Container Registry (ICR),
* Access to an OpenShift v4.7 cluster, this tutorial uses IBM Cloud Red Hat OpenShift Kubernetes Service (ROKS),
* OpenShift Pipelines must be installed,
* Connect to OpenShift,
* Create a new project,
* Clone the git repository for this tutorial, containing the Tekton resources,

## IBM Cloud Shell

Access to a client terminal like [IBM Cloud Shell](https://cloud.ibm.com/shell).

## IBM Cloud Container Registry (ICR)


<!--
Make sure a Route to the OpenShift registry service exists,

```console
oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
```
-->

## IBM Cloud Red Hat OpenShift Kubernetes Service (ROKS)

Connect to your OpenShift cluster from the terminal, follow [these instructions](https://ibm.github.io/workshop-setup/ROKS/).

## OpenShift Pipelines

Before you start the tutorial you must have access to an OpenShift environment with OpenShift Pipelines using Tekton. 

```bash
$ oc get operators
NAME                                                  AGE
openshift-pipelines-operator-rh.openshift-operators   12d
```

To install OpenShift Pipelines, follow [this guide on installing OpenShift Pipelines](https://github.com/openshift/pipelines-tutorial/blob/master/install-operator.md).

## Connect to OpenShift

To connect to your OpenShift cluster, follow the instructions [here](https://ibm.github.io/workshop-setup/ROKS/).

## Create a New Project

Create a new project,

```bash
oc new-project tekton101lab
```

Or change the current project to `tekton101lab`,

```bash
oc project tekton101lab
```

## Clone Git Repository

Clone the source code for this workshop to your client in order to edit some of the yaml files before applying them to your cluster. Check out the `beta-update` branch after cloning.

```bash
git clone https://github.com/IBM/tekton-tutorial-openshift
cd tekton-tutorial-openshift
```

## Next

Next, go to [Create a Task to Clone a Git Repository](1_clone-git-repo.md).
