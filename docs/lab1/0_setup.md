# Setup

## Prerequisites

Access to a client terminal like [IBM Cloud Shell](https://cloud.ibm.com/shell).

Before you start the tutorial you must have access to an OpenShift environment with OpenShift Pipelines using Tekton. To install OpenShift Pipelines, follow [this guide on installing OpenShift Pipelines](https://github.com/openshift/pipelines-tutorial/blob/master/install-operator.md).

Connect to your OpenShift cluster from the terminal, follow [these instructions](https://ibm.github.io/workshop-setup/ROKS/).

Create a new project,

```bash
oc new-project tekton101lab
```

Or change the current project to `tekton101lab`,

```bash
oc project tekton101lab
```

Make sure a Route to the OpenShift registry service exists,

```console
oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
```

Clone the source code for this workshop to your client in order to edit some of the yaml files before applying them to your cluster. Check out the `beta-update` branch after cloning.

```bash
git clone https://github.com/IBM/tekton-tutorial-openshift
cd tekton-tutorial-openshift
```
