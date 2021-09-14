# Setup

## Prerequisites

* IBM Cloud Account,
* Access to IBM Cloud shell,
* Target a Region,
* Git repository containing Tekton resources,
* IBM Cloud Container Registry (ICR),
* Access to an OpenShift v4.7 cluster, this tutorial uses IBM Cloud Red Hat OpenShift Kubernetes Service (ROKS),
* OpenShift Pipelines must be installed,
* Create a New Project,

## IBM Cloud Account

Access your IBM Cloud account. Open a browser and go to [https://cloud.ibm.com](https://cloud.ibm.com). If you need to create a new IBM Cloud account, go [here](https://ibm.github.io/workshop-setup/NEWACCOUNT/).

## IBM Cloud Shell

In your IBM Cloud account, open a new instance of [IBM Cloud Shell](https://cloud.ibm.com/shell).

## Target a Region

Target the same region as your OpenShift cluster. In this tutorial, I am using a cluster in the region `us-south`.

```bash
REGION=us-south
ibmcloud target -r $REGION
```

## Git repository containing Tekton resources

Clone the source code for this workshop to your client terminal in order to edit some of the yaml files before applying them to your cluster. Change into the directory containing the resources.

```bash
git clone https://github.com/IBM/tekton-tutorial-openshift
cd tekton-tutorial-openshift
```

## IBM Cloud Container Registry (ICR)

Your IBM Cloud account includes a registry called [IBM Cloud Container Registry (ICR)](https://cloud.ibm.com/docs/Registry?topic=Registry-registry_overview), a multi-tenant, highly available, scalable, and encrypted private image registry. You can access your registry at [https://cloud.ibm.com/registry/overview](https://cloud.ibm.com/registry/overview).

The IBM Cloud Shell already has a Container Registry CLI plugin installed. Set the region and confirm the region is set for the registry on your account.

```bash
$ ibmcloud cr 
$ ibmcloud cr region-set $REGION
The region is set to 'us-south', the registry is 'us.icr.io'.
OK

$ ibmcloud cr api                         
Registry API endpoint   https://us.icr.io/api   
OK
```

Create environment variables for the registry route and a namespace in your registry. Both variables will be used in the rest of the workshop. Create the registry namespace.

```bash
$ ibmcloud cr namespace-list
Listing namespaces for account 'IBM Client Developer Advocacy' in registry 'us.icr.io'...
...

$ REGISTRY_ROUTE=us.icr.io
$ NAMESPACE=tekton101lab
$ ibmcloud cr namespace-add $NAMESPACE
```

## IBM Cloud Red Hat OpenShift Kubernetes Service (ROKS)

Connect to your OpenShift cluster from the terminal using the login command from the OpenShift console, which should look similar to the following command,

```bash
oc login --token=sha256~ZBMKw9VAayhdnyANaHvjJeXDiGwA7Fsr5gtLKj3-ehE --server=https://d107-f.us-south.containers.cloud.ibm.com:30271
```

If you need to retrieve the login command, follow [these instructions](https://ibm.github.io/workshop-setup/ROKS/).

## OpenShift Pipelines

Before you start the tutorial you must have access to an OpenShift environment with OpenShift Pipelines using Tekton.

```bash
$ oc get operators
NAME                                                  AGE
openshift-pipelines-operator-rh.openshift-operators   12d
```

To install OpenShift Pipelines, follow [this guide on installing Red Hat OpenShift Pipelines](https://ibm.github.io/workshop-setup/PIPELINES/).

## Create a New Project

Create a new project or namespace, you can use the same name as for the container registry, though they are not related.

```bash
$ NAMESPACE=tekton101lab
$ oc new-project $NAMESPACE

Now using project "tekton101lab" on server "https://d107-f.us-south.containers.cloud.ibm.com:30271".
You can add applications to this project with the 'new-app' command. For example, try:
    oc new-app rails-postgresql-example
to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:
    kubectl create deployment hello-node --image=k8s.gcr.io/serve_hostname
```

Or change the current project to `tekton101lab`,

```bash
oc project tekton101lab
```

## Next

Next, go to [Create a Task to Clone a Git Repository](1_clone-git-repo.md).
