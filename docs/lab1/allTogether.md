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

REGISTRY_ROUTE=us.icr.io
NAMESPACE=advowork
```

Since everybody in the lab will be pushing to the same namespace in the registry, we need a way to differentiate each other's images. We can do this by adding your name to the image tag. In the following command, substitute `<name>` with your first name.

```sh
NAME=<name>
```

For example,

```sh
$ NAME=oliver
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

To install OpenShift Pipelines, run the following command:

```sh
oc apply -f src/pipelinesOperatorSubscription.yaml
```

Optional: You can also install OpenShift Pipelines through the OpenShift Console by following [this guide on installing Red Hat OpenShift Pipelines](https://ibm.github.io/workshop-setup/PIPELINES/).

## Create a New Project

Create a new project or namespace, you can use the same name as for the container registry, though they are not related.

```bash
$ oc new-project tekton101lab

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

# 1. Create a Task to Clone the Git Repository

The first thing that the pipeline needs is a task to clone the Git repository with the source code for the image that the pipeline is building. This is such a common function that you don't need to write this task yourself. Tekton provides a library of reusable tasks called the [Tekton Hub](https://hub.tekton.dev).
The library provides a `git-clone` task which is described [here](https://hub.tekton.dev/tekton/task/git-clone).

The `git-clone` task is reproduced partially below.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: git-clone
  labels:
    app.kubernetes.io/version: "0.4"
  annotations:
    tekton.dev/pipelines.minVersion: "0.21.0"
    tekton.dev/categories: Git
    tekton.dev/tags: git
    tekton.dev/displayName: "git clone"
spec:
  description: >-
    These Tasks are Git tasks to work with repositories used by other tasks
    in your Pipeline.
  workspaces:
    - name: output
      description: The git repo will be cloned onto the volume backing this Workspace.
    - name: ssh-directory
      optional: true
      description: |
        A .ssh directory with private key, known_hosts, config, etc.
    - name: basic-auth
      optional: true
      description: |
        A Workspace containing a .gitconfig and .git-credentials file.
  params:
    - name: url
      description: Repository URL to clone from.
      type: string
    - name: revision
      description: Revision to checkout. (branch, tag, sha, ref, etc...)
      type: string
      default: ""
    - name: submodules
      description: Initialize and fetch git submodules.
      type: string
      default: "true"
    - name: depth
      description: Perform a shallow clone, fetching only the most recent N commits.
      type: string
      default: "1"
    - name: sslVerify
      description: Set the `http.sslVerify` global git config. Setting this to `false` is not advised unless you are sure that you trust your git remote.
      type: string
      default: "true"
    - name: subdirectory
      description: Subdirectory inside the `output` Workspace to clone the repo into.
      type: string
      default: ""
    - name: deleteExisting
      description: Clean out the contents of the destination directory if it already exists before cloning.
      type: string
      default: "true"
  results:
    - name: commit
      description: The precise commit SHA that was fetched by this Task.
    - name: url
      description: The precise URL that was fetched by this Task.
  steps:
    - name: clone
      image: "$(params.gitInitImage)"
      env:
      - name: HOME
        value: "$(params.userHome)"
      - name: PARAM_URL
        value: $(params.url)
      - name: PARAM_REVISION
        value: $(params.revision)
      - name: PARAM_SUBMODULES
        value: $(params.submodules)
      - name: PARAM_DEPTH
        value: $(params.depth)
      - name: PARAM_SSL_VERIFY
        value: $(params.sslVerify)
      - name: PARAM_SUBDIRECTORY
        value: $(params.subdirectory)
      - name: PARAM_DELETE_EXISTING
        value: $(params.deleteExisting)
      script: |
        #!/usr/bin/env sh
        set -eu

        if [ "${PARAM_VERBOSE}" = "true" ] ; then
          set -x
        fi

        if [ "${WORKSPACE_BASIC_AUTH_DIRECTORY_BOUND}" = "true" ] ; then
          cp "${WORKSPACE_BASIC_AUTH_DIRECTORY_PATH}/.git-credentials" "${PARAM_USER_HOME}/.git-credentials"
          cp "${WORKSPACE_BASIC_AUTH_DIRECTORY_PATH}/.gitconfig" "${PARAM_USER_HOME}/.gitconfig"
          chmod 400 "${PARAM_USER_HOME}/.git-credentials"
          chmod 400 "${PARAM_USER_HOME}/.gitconfig"
        fi

        if [ "${WORKSPACE_SSH_DIRECTORY_BOUND}" = "true" ] ; then
          cp -R "${WORKSPACE_SSH_DIRECTORY_PATH}" "${PARAM_USER_HOME}"/.ssh
          chmod 700 "${PARAM_USER_HOME}"/.ssh
          chmod -R 400 "${PARAM_USER_HOME}"/.ssh/*
        fi

        CHECKOUT_DIR="${WORKSPACE_OUTPUT_PATH}/${PARAM_SUBDIRECTORY}"

        cleandir() {
          # Delete any existing contents of the repo directory if it exists.
          #
          # We don't just "rm -rf ${CHECKOUT_DIR}" because ${CHECKOUT_DIR} might be "/"
          # or the root of a mounted volume.
          if [ -d "${CHECKOUT_DIR}" ] ; then
            # Delete non-hidden files and directories
            rm -rf "${CHECKOUT_DIR:?}"/*
            # Delete files and directories starting with . but excluding ..
            rm -rf "${CHECKOUT_DIR}"/.[!.]*
            # Delete files and directories starting with .. plus any other character
            rm -rf "${CHECKOUT_DIR}"/..?*
          fi
        }

        if [ "${PARAM_DELETE_EXISTING}" = "true" ] ; then
          cleandir
        fi

        test -z "${PARAM_HTTP_PROXY}" || export HTTP_PROXY="${PARAM_HTTP_PROXY}"
        test -z "${PARAM_HTTPS_PROXY}" || export HTTPS_PROXY="${PARAM_HTTPS_PROXY}"
        test -z "${PARAM_NO_PROXY}" || export NO_PROXY="${PARAM_NO_PROXY}"

        /ko-app/git-init \
          -url="${PARAM_URL}" \
          -revision="${PARAM_REVISION}" \
          -refspec="${PARAM_REFSPEC}" \
          -path="${CHECKOUT_DIR}" \
          -sslVerify="${PARAM_SSL_VERIFY}" \
          -submodules="${PARAM_SUBMODULES}" \
          -depth="${PARAM_DEPTH}" \
          -sparseCheckoutDirectories="${PARAM_SPARSE_CHECKOUT_DIRECTORIES}"
        cd "${CHECKOUT_DIR}"
        RESULT_SHA="$(git rev-parse HEAD)"
        EXIT_CODE="$?"
        if [ "${EXIT_CODE}" != 0 ] ; then
          exit "${EXIT_CODE}"
        fi
        printf "%s" "${RESULT_SHA}" > "$(results.commit.path)"
        printf "%s" "${PARAM_URL}" > "$(results.url.path)"
```

A `Task` can have one or more `Steps`. Each step defines an image to run to perform the function of the step. The `git-clone` task has only one step called `clone` that uses a Tekton-provided container to clone a Git repo.

A task can have `Parameters`. Parameters help to make a task reusable. This task accepts many parameters such as:

* `url`, the Repository URL to clone from,
* `revision`, the revision to check out (branch, tag, sha, ref, etc),
* `subdirectory`, a subdirectory inside the output Workspace to clone the repo into,
* `deleteExisting`, clean existing destination directory if it already exists before cloning, and more.

Parameters can have default values provided by the task or the values can be provided by the `Pipeline` and `PipelineRun` resources as we will see later. Steps can reference parameter values by using the syntax `$(params.name)` where `name` is the name of the parameter. For example the step uses `$(params.url)` to reference the `url` parameter value.

The task requires a `Workspace` where the clone is stored. From the point of view of the task, a workspace provides a file system path where it can read or write data.
Steps can reference the path using the syntax `$(workspaces.name.path)` where `name` is the name of the workspace.
We'll see later how the workspace becomes associated with a storage volume.

Though the task has been defined and is available from the Tekton Hub, you still need to apply and create the Task in your OpenShift cluster. Apply the file to your cluster to create the task in your namespace,

```bash
$ oc apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.3/git-clone.yaml
task.tekton.dev/git-clone created

$ oc get tasks
NAME        AGE
git-clone   3m9s
```

# 2. Create a Task to Build and Upload Container Image using Kaniko

The next task that the pipeline needs is a task that builds a docker image and pushes it to a container registry. The catalog provides a `kaniko` task which does this using Google's `kaniko` tool. The task is described [here](https://hub.tekton.dev/tekton/task/kaniko).

The task is reproduced below.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: kaniko
  labels:
    app.kubernetes.io/version: "0.2"
  annotations:
    tekton.dev/pipelines.minVersion: "0.12.1"
    tekton.dev/categories: Image Build
    tekton.dev/tags: image-build
spec:
  description: >-
    This Task builds source into a container image using Google's kaniko tool.

    Kaniko doesn't depend on a Docker daemon and executes each
    command within a Dockerfile completely in userspace. This enables
    building container images in environments that can't easily or
    securely run a Docker daemon, such as a standard Kubernetes cluster.

  params:
  - name: IMAGE
    description: Name (reference) of the image to build.
  - name: DOCKERFILE
    description: Path to the Dockerfile to build.
    default: ./Dockerfile
  - name: CONTEXT
    description: The build context used by Kaniko.
    default: ./
  - name: EXTRA_ARGS
    default: ""
  - name: BUILDER_IMAGE
    description: The image on which builds will run (default is v1.5.1)
    default: gcr.io/kaniko-project/executor:v1.5.1@sha256:c6166717f7fe0b7da44908c986137ecfeab21f31ec3992f6e128fff8a94be8a5
  workspaces:
  - name: source
  results:
  - name: IMAGE-DIGEST
    description: Digest of the image just built.
  steps:
  - name: build-and-push
    workingDir: $(workspaces.source.path)
    image: $(params.BUILDER_IMAGE)
    # specifying DOCKER_CONFIG is required to allow kaniko to detect docker credential
    # https://github.com/tektoncd/pipeline/pull/706
    env:
    - name: DOCKER_CONFIG
      value: /tekton/home/.docker
    args:
    - $(params.EXTRA_ARGS)
    - --dockerfile=$(params.DOCKERFILE)
    - --context=$(workspaces.source.path)/$(params.CONTEXT)  # The user does not need to care the workspace and the source.
    - --destination=$(params.IMAGE)
    - --oci-layout-path=$(workspaces.source.path)/$(params.CONTEXT)/image-digest
    # kaniko assumes it is running as root, which means this example fails on platforms
    # that default to run containers as random uid (like OpenShift). Adding this securityContext
    # makes it explicit that it needs to run as root.
    securityContext:
      runAsUser: 0
  - name: write-digest
    workingDir: $(workspaces.source.path)
    image: gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/imagedigestexporter:v0.16.2
    # output of imagedigestexport [{"key":"digest","value":"sha256:eed29..660","resourceRef":{"name":"myrepo/myimage"}}]
    command: ["/ko-app/imagedigestexporter"]
    args:
    - -images=[{"name":"$(params.IMAGE)","type":"image","url":"$(params.IMAGE)","digest":"","OutputImageDir":"$(workspaces.source.path)/$(params.CONTEXT)/image-digest"}]
    - -terminationMessagePath=$(params.CONTEXT)/image-digested
    securityContext:
      runAsUser: 0
  - name: digest-to-results
    workingDir: $(workspaces.source.path)
    image: docker.io/stedolan/jq@sha256:a61ed0bca213081b64be94c5e1b402ea58bc549f457c2682a86704dd55231e09
    script: |
      cat $(params.CONTEXT)/image-digested | jq '.[0].value' -rj | tee /tekton/results/IMAGE-DIGEST
```

You can see that this task needs a workspace as well.

```yml
workspaces:
  - name: source
```

This workspace has the source to be build. The pipeline will provide the same workspace that it used for the `git-clone` task.

The `kaniko` task also uses a feature called `results`. A result is a value produced by a task which can then be used as a parameter value to other tasks. This task declares a result named `IMAGE-DIGEST` which it sets to the digest of the built image. A task sets a result by writing it to a file named `/tekton/results/name` where `name` refers to the name of the result, in this case `IMAGE-DIGEST`. We will see later how the pipeline uses this result.

```yml
results:
  - name: IMAGE-DIGEST
```

We will cover how the task authenticates to the image repository for permission to push the image later.

First, apply the file to your cluster and create the `kaniko` task in your namespace.

```bash
$ oc apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/kaniko/0.2/kaniko.yaml
task.tekton.dev/kaniko created

$ oc get tasks
NAME        AGE
git-clone   48m
kaniko      3m9s
```

# 3. Create a Task to Deploy an Image to OpenShift

The final function that the pipeline needs is a task that deploys a docker image to a Kubernetes cluster.

Below is a Tekton task that does this.
You can find this yaml file at [tekton/tasks/deploy-using-kubectl.yaml](https://github.com/IBM/tekton-tutorial-openshift/blob/master/tekton/tasks/deploy-using-kubectl.yaml).

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: deploy-using-kubectl
spec:
  workspaces:
    - name: git-source
      description: The git repo
  params:
    - name: pathToYamlFile
      description: The path to the yaml file to deploy within the git source
    - name: imageUrl
      description: Image name including repository
    - name: imageTag
      description: Image tag
      default: "latest"
    - name: imageDigest
      description: Digest of the image to be used.
  steps:
    - name: update-yaml
      image: cgr.dev/chainguard/wolfi-base
      command: ["sed"]
      args:
        - "-i"
        - "-e"
        - "s;__IMAGE__;$(params.imageUrl):$(params.imageTag);g"
        - "-e"
        - "s;__DIGEST__;$(params.imageDigest);g"
        - "$(workspaces.git-source.path)/$(params.pathToYamlFile)"
    - name: run-kubectl
      image: lachlanevenson/k8s-kubectl
      command: ["kubectl"]
      args:
        - "apply"
        - "-f"
        - "$(workspaces.git-source.path)/$(params.pathToYamlFile)"
```

This task has two steps.

1. The first step runs the `sed` command in an Wolfi container and updates the yaml file used for deployment. This step replaces the default image reference `__IMAGE__` by the image that was built by the `kaniko` task `__DIGEST__`. This step requires the yaml file to have two character strings, `__IMAGE__` and `__DIGEST__`, which are substituted with parameter values.

2. The second step runs the `kubectl` command using a `k8s-kubectl` container image by Lachlan Evenson, to apply the yaml file to the same cluster where the pipeline is running.

As was the case in the `git-clone` and `kaniko` tasks, this task makes use of parameters in order to make the task as reusable as possible. It also needs the workspace to get the deployment yaml file.

How the task authenticates to the cluster for permission to apply the resource(s) in the yaml file, is covered later on in the tutorial.

Apply the file to your cluster to create the task `deploy-using-kubectl` in your namespace.

```bash
$ oc apply -f tekton/tasks/deploy-using-kubectl.yaml
task.tekton.dev/deploy-using-kubectl created

$ oc get tasks
NAME                   AGE
deploy-using-kubectl   54s
git-clone              3h3m
kaniko                 138m
```

# 4. Create a Pipeline

Now we created all the tasks that we need , we need to create a pipeline definition `build-and-deploy-pipeline` that defines how to run our tasks. Below is a Tekton pipeline that runs the tasks we created above. You can find this yaml file at [https://github.com/IBM/tekton-tutorial-openshift/blob/master/tekton/pipeline/build-and-deploy-pipeline.yaml](https://github.com/IBM/tekton-tutorial-openshift/blob/master/tekton/pipeline/build-and-deploy-pipeline.yaml).

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: build-and-deploy-pipeline
spec:
  workspaces:
    - name: git-source
      description: The git repo
  params:
    - name: gitUrl
      description: Git repository url
    - name: pathToContext
      description: The path to the build context, used by Kaniko - within the workspace
      default: src
    - name: pathToYamlFile
      description: The path to the yaml file to deploy within the git source
    - name: imageUrl
      description: Image name including repository
    - name: imageTag
      description: Image tag
      default: "latest"
  tasks:
    - name: clone-repo
      taskRef:
        name: git-clone
      workspaces:
        - name: output
          workspace: git-source
      params:
        - name: url
          value: "$(params.gitUrl)"
        - name: subdirectory
          value: "."
        - name: deleteExisting
          value: "true"
    - name: build-and-push-image
      taskRef:
        name: kaniko
      runAfter:
        - clone-repo
      workspaces:
        - name: source
          workspace: git-source
      params:
        - name: CONTEXT
          value: $(params.pathToContext)
        - name: IMAGE
          value: $(params.imageUrl):$(params.imageTag)
    - name: deploy-to-cluster
      taskRef:
        name: deploy-using-kubectl
      workspaces:
        - name: git-source
          workspace: git-source
      params:
        - name: pathToYamlFile
          value: $(params.pathToYamlFile)
        - name: imageUrl
          value: $(params.imageUrl)
        - name: imageTag
          value: $(params.imageTag)
        - name: imageDigest
          value: $(tasks.build-and-push-image.results.IMAGE-DIGEST)
```

A `Pipeline` resource contains a list of tasks to run. Each pipeline task is assigned a `name` within the pipeline;  here they are `clone-repo`, `build-and-push-image`, and `deploy-using-kubectl`. The pipeline configures each task via the task's parameters. You can choose whether to expose a task parameter as a pipeline parameter, set the value directly, or let the value
default inside the task (if it's an optional parameter).  

For example, our pipeline `build-and-deploy-pipeline` defines a parameter `pathToContext` that defaults to `src`. If you run the pipeline you can pass a value for this parameter and override the default. In the `build-and-push-image` task of the pipeline, a `CONTEXT` parameter is defined, which corresponds to the `CONTEXT` parameter that is defined in the `kaniko` task. The value of this parameter in the kaniko task will be set to the value that our pipeline parameter `pathToContext` is set to.

However, the parameter `DOCKERFILE` that is defined in our `kaniko` task and there defaults to `./Dockerfile`, is not exposed in the pipeline under the `build-and-push-image` task that references the `kaniko` task. So the pipeline is not overriding the value of the `DOCKERFILE` parameter, and thus it defaults to the value `./Dockerfile`.

Our `build-and-deploy-pipeline` pipeline also shows how to take the result of one task and pass it to another task. You saw earlier that the `kaniko` task defines a result named `IMAGE-DIGEST` that holds the digest of the built image.

```yaml
results:
  - name: IMAGE-DIGEST
```

The pipeline passes that value to the `deploy-using-kubectl` task by using the syntax `$(tasks.build-and-push-image.results.IMAGE-DIGEST)` where `build-and-push-image` is the name used in the pipeline to run the `kaniko` task.

By default Tekton assumes that pipeline tasks can be executed concurrently. In this pipeline each pipeline task depends on the previous one so they must be executed sequentially. One way that dependencies between pipeline tasks can be expressed is by using the `runAfter` key. It specifies that the task must run after the given list of tasks has completed. In this example, the pipeline specifies that the `build-and-push-image` pipeline task must run after the `clone-repo` pipeline task.

The `deploy-using-kubectl` task must run after the `build-and-push-image` pipeline task, but it doesn't need to specify the `runAfter` key. This is because it references a task result from the `build-and-push-image` pipeline task and Tekton is smart enough to figure out that this means it must run after that task.

Apply the file to your cluster to create the pipeline in your namespace.

```bash
$ oc apply -f tekton/pipeline/build-and-deploy-pipeline.yaml
pipeline.tekton.dev/build-and-deploy-pipeline created

$ oc get pipelines
NAME                        AGE
build-and-deploy-pipeline   17d
```

# 5. Define a Service Account

Before running the pipeline, we need to set up a service account. In OpenShift, a service account is associated with a username and can be granted roles to control access to protected resources. A service account can use secrets containing credentials for authentication along with RBAC-related resources for permission to create and modify relevant Kubernetes resources.

## Set up IBM Cloud Container Registry (ICR) as a private registry on Red Hat OpenShift

Red Hat OpenShift on IBM Cloud clusters are also set up by default with an internal registry that stores images locally in your cluster. You can use either the internal registry on OpenShift or the private IBM Cloud Container Registry (ICR), in combination or separately. For instance, you could create an ImageStream on your cluster, pull copies of images in ICR to the internal private registry, and in a deployment use the image from the internal private registry.

In this workshop, we use only the private IBM Cloud Container Registry (ICR). To access ICR, you need to [set up access to IBM Cloud Container Registry (ICR) as a private registry on Red Hat OpenShift](https://cloud.ibm.com/docs/Registry?topic=Registry-registry_rhos).

Create a new IBM Cloud API Key or use an existing IBM Cloud API Key

```bash
$ USERNAME=<your username>

$ ibmcloud iam api-key-create $USERNAME-tekton-apikey -d "apikey for tekton task in openshift" --file apikey-tekton.json
Creating API key user1-tekton-apikey under e65910fa61ce9072d64902d03f3d4774 as user1@email.com...OK
API key user1-tekton-apikey was created
Successfully save API key information to apikey-tekton.json

$ IBMCLOUD_APIKEY=$(cat apikey-tekton.json | jq -r ".apikey")
$ echo $IBMCLOUD_APIKEY

$ oc create secret generic ibm-cr-push-secret --type="kubernetes.io/basic-auth" --from-literal=username=iamapikey --from-literal=password=$IBMCLOUD_APIKEY

secret/ibm-cr-push-secret created

$ oc annotate secret ibm-cr-push-secret tekton.dev/docker-0=us.icr.io

secret/ibm-cr-push-secret annotated

$ EMAIL=user1v@email.com
$ oc create secret docker-registry ibm-cr-pull-secret --docker-server=$REGISTRY_ROUTE --docker-username=iamapikey --docker-password=$IBMCLOUD_APIKEY --docker-email=$EMAIL
```

Next, create a service account using the following yaml file at [tekton/pipeline-account.yaml](https://github.com/IBM/tekton-tutorial-openshift/blob/master/tekton/pipeline-account.yaml).

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipeline-account
secrets:
- name: ibm-cr-push-secret

---

apiVersion: v1
kind: Secret
metadata:
  name: kube-api-secret
  annotations:
    kubernetes.io/service-account.name: pipeline-account
type: kubernetes.io/service-account-token

---

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pipeline-role
rules:
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "create", "update", "patch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "create", "update", "patch"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pipeline-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pipeline-role
subjects:
- kind: ServiceAccount
  name: pipeline-account
```

This yaml creates the following Kubernetes resources:

* A ServiceAccount named `pipeline-account`. The service account references the annotated `ibm-cr-push-secret` secret so that the pipeline authenticates to your private container registry when it pushes and pulls a container image from the private ICR.

* A Secret named `kube-api-secret` which contains a token (generated by Kubernetes) for accessing the Kubernetes API. This allows the pipeline to use `kubectl` or `oc` to talk to your cluster. In OpenShift, a controller loop ensures a Secret with an API token exists for ServiceAccounts. To create our own additional API token for our ServiceAccount, we need to create a Secret of type `kubernetes.io/service-account-token` with an annotation referencing the ServiceAccount. The controller will update it with a generated token.

* A Role named `pipeline-role` which provides the Resource-Based Access Control (RBAC) permissions needed for this pipeline to create and modify Kubernetes resources, respectively services and deployments.

* And a RoleBinding named `pipeline-role-binding`, which binds the Role to your Service Account.

Apply the file to your cluster to create the service account and related resources.

```bash
$ oc apply -f tekton/pipeline-account.yaml
serviceaccount/pipeline-account created
secret/kube-api-secret created
role.rbac.authorization.k8s.io/pipeline-role created
rolebinding.rbac.authorization.k8s.io/pipeline-role-binding created
```

Our pipeline-account is now authorized to get, create, update and patch services and deployments.

# 6. Create a PipelineRun

You have defined and created a reusable Pipeline and Task resources for building and deploying an image. It is now time to define how to run the pipeline by creating a Tekton PipelineRun resource. You can find the yaml file for the PipelineRun at [tekton/run/picalc-pipeline-run.yaml](https://github.com/IBM/tekton-tutorial-openshift/blob/master/tekton/run/picalc-pipeline-run.yaml).

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: tekton101lab-pipelinerun-
spec:
  pipelineRef:
    name: build-and-deploy-pipeline
  params:
    - name: gitUrl
      value: https://github.com/ibm/tekton-tutorial-openshift/
    - name: pathToYamlFile
      value: kubernetes/picalc.yaml
    - name: imageUrl
      value: <REGISTRY>/<NAMESPACE>/picalc
    - name: imageTag
      value: <NAME>-v1
  serviceAccountName: pipeline-account
  workspaces:
    - name: git-source
      persistentVolumeClaim:
        claimName: picalc-source-pvc
```

Although this file is small there is a lot going on here.  Let's break it down from top to bottom:

* The PipelineRun does not have a fixed name. It uses `generateName` to generate a name each time it is created using a prefix `tekton101lab-pipelinerun` and a hyphen. This is because a particular PipelineRun resource executes the pipeline only once. If you want to run the pipeline again, you cannot modify an existing PipelineRun resource to request it to re-run -- you must technically create a new PipelineRun resource. While you could use `name` to assign a unique name to your PipelineRun each time you create one, it is much easier to use `generateName` and a custom name appended by a hyphen, e.g. `tekton101lab-pipelinerun-`.

* The Pipeline resource is identified under the `pipelineRef` key and refers to the Pipeline resource we created earlier.

* Parameters exposed by the pipeline are set to specific values such as the Git repository to clone, the image to build, and the yaml file to deploy. This example builds a [go program that calculates an approximation of pi](https://github.com/IBM/tekton-tutorial-openshift/blob/master/src/picalc.go) called `picalc`. The source includes a [Dockerfile](src/Dockerfile) which runs tests, compiles the code, and builds an image for execution.

We have to edit the `picalc-pipeline-run.yaml` file to substitute the values of `<REGISTRY>` and `<NAMESPACE>` in the file with the information from your private container registry and the registry namespace. We found and set values for `REGISTRY_ROUTE` and `NAMESPACE` in the [setup](0_setup.md) section of the workshop.

Check the values for `REGISTRY_ROUTE` and `NAMESPACE` are set.

```bash
echo $REGISTRY_ROUTE
echo $NAMESPACE
echo $NAME
```

This should set the `imageUrl` to `us.icr.io/tekton101lab/picalc`. Run the following `sed` commands,

```bash
sed -i "s/<REGISTRY>/$REGISTRY_ROUTE/g" tekton/run/picalc-pipeline-run.yaml
sed -i "s/<NAMESPACE>/$NAMESPACE/g" tekton/run/picalc-pipeline-run.yaml
sed -i "s/<NAME>/$NAME/g" tekton/run/picalc-pipeline-run.yaml

cat tekton/run/picalc-pipeline-run.yaml
```

* The service account named `pipeline-account` which we created earlier is specified to provide the credentials needed for the pipeline to run successfully.

* The workspace used by the pipeline to clone the Git repository is mapped to a persistent volume claim which is a request for a storage volume.

Before you run the pipeline for the first time, you must create a persistent volume claim for the workspace.

```bash
$ oc create -f tekton/picalc-pipeline-pvc.yaml
persistentvolumeclaim/picalc-source-pvc created
```

The persistent volume claim requests Kubernetes to obtain a storage volume. Since each PipelineRun references the same claim and thus the same volume, this means the PipelineRun can only be run serially to avoid conflicting use of the volume.

**Note** new [funtionality in Tekton](https://github.com/tektoncd/pipeline/pull/2326) was merged to allow each PipelineRun to create its own persistent volume claim and thus use its own volume. This workshop will be updated to include this new functionality soon.

Check that the persistent volume claim is bound before continuing.

```bash
$ oc get pvc picalc-source-pvc
NAME                STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS      AGE
picalc-source-pvc   Pending                                      ibmc-block-gold   23s

$ oc get pvc picalc-source-pvc
NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
picalc-source-pvc   Bound    pvc-2ec590d9-13dc-4dbe-b9ee-473ca6e74c91   20Gi       RWO            ibmc-block-gold   78s
```

The PersistentVolumeClaim with storage class `ibmc-block-gold` uses dynamic provisioning to create the PersistentVolume for the PersistentVolumeClaim.

```bash
$ oc get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                             STORAGECLASS     REASON   AGE
pvc-15a4b091-f6f9-42f6-b3c3-a354a440acd0   20Gi       RWO            Delete           Bound    tekton101lab/picalc-source-pvc                                              7m23s
pvc-f2618698-d7b7-4ba6-8b4b-2e162d55e6c8   100Gi      RWX            Delete           Bound    openshift-image-registry/image-registry-storage   ibmc-file-gold            22d
```

# 7. Run the Pipeline

All the pieces are in place to execute the pipelinerun by creating the PipelineRun resource in your cluster.

```bash
$ oc create -f tekton/run/picalc-pipeline-run.yaml
pipelinerun.tekton.dev/picalc-pr-c7hsb created
```

Note that we're using `oc create` here instead of `oc apply`. As mentioned before, a given PipelineRun resource can run a pipeline only once, so you need to create a new PipelineRun each time you want to run the Pipeline. `kubectl` or `oc` will create the generated name of the PipelineRun resource.

View the status of our pipeline run,

```bash
$ oc get pipelinerun
NAME              SUCCEEDED   REASON    STARTTIME   COMPLETIONTIME
picalc-pr-dqwqb   Unknown     Running   5s
```

The pipeline will be successful when `SUCCEEDED` is `True`.

```bash
$ oc get pipelinerun
NAME              SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
picalc-pr-x5qf9   True        Succeeded   2m58s       12s
```

Check the status of the Kubernetes deployment.  It should be ready.

```bash
$ oc get deploy picalc
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
picalc   1/1     1            1           9m
```

You can curl the application using its NodePort service.
First display the nodes and choose one of the node's external IP addresses.
Then display the service to get its NodePort.

```bash
$ kubectl get nodes -o wide
NAME           STATUS   ROLES    AGE     VERSION       INTERNAL-IP    EXTERNAL-IP      OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
10.221.22.11   Ready    <none>   7d23h   v1.16.8+IKS   10.221.22.11   150.238.236.26   Ubuntu 18.04.4 LTS   4.15.0-96-generic   containerd://1.3.3
10.221.22.49   Ready    <none>   7d23h   v1.16.8+IKS   10.221.22.49   150.238.236.21   Ubuntu 18.04.4 LTS   4.15.0-96-generic   containerd://1.3.3

$ kubectl get svc picalc
NAME     TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
picalc   NodePort   172.21.199.71   <none>        8080:30925/TCP   9m

$ EXTERNAL_IP=150.238.236.26
$ NODE_PORT=30925
$ curl $EXTERNAL_IP:$NODE_PORT?iterations=20000000
3.1415926036
```

## Debugging a failed PipelineRun

Let's take a look at what a PipelineRun failure would look like.
Edit the PipelineRun yaml and change the gitUrl parameter to a non-existent Git repository to force a failure.
Then create a new PipelineRun and describe it after letting it run for a minute or two.

```bash
$ kubectl create -f tekton/picalc-pipeline-run.yaml
pipelinerun.tekton.dev/picalc-pr-sk7md created

$ tkn pipelinerun describe picalc-pr-sk7md
Name:              picalc-pr-sk7md
Namespace:         default
Pipeline Ref:      build-and-deploy-pipeline
Service Account:   pipeline-account

Status

STARTED         DURATION     STATUS
2 minutes ago   41 seconds   Failed

Message

TaskRun picalc-pr-sk7md-clone-repo-8gs25 has failed (&#34;step-clone&#34; exited with code 1 (image: &#34;gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/git-init@sha256:bee98bfe6807e8f4e0a31b4e786fd1f7f459e653ed1a22b1a25999f33fa9134a&#34;); for logs run: kubectl -n default logs picalc-pr-sk7md-clone-repo-8gs25-pod-v7fg8 -c step-clone)

Resources

 No resources

Params

 NAME             VALUE
 gitUrl           https://github.com/IBM/tekton-tutorial-not-there
 pathToYamlFile   kubernetes/picalc.yaml
 imageUrl         us.icr.io/gregd/picalc
 imageTag         1.0

Taskruns

 NAME                               TASK NAME    STARTED         DURATION     STATUS
 picalc-pr-sk7md-clone-repo-8gs25   clone-repo   2 minutes ago   41 seconds   Failed
```

The output tells us that the `clone-repo` pipeline task failed.  The `Message` also tells us how to get the logs from the pod which was used to run the task:

```bash
for logs run: kubectl -n default logs picalc-pr-sk7md-clone-repo-8gs25-pod-v7fg8 -c step-clone
```

If you run that `kubectl logs` command you will see that there is a failure trying to fetch the non-existing Git repository.

An even easier way to get the logs from a PipelineRun is to use the `tkn` CLI.

```bash
tkn pipelinerun logs picalc-pr-sk7md-clone-repo-8gs25-pod-v7fg8 -t clone-repo
```

If you omit the `-t` flag then the command will get the logs for all pipeline tasks that executed.

You can also get the logs for the last PipelineRun for a particular Pipeline using this command:

```bash
tkn pipeline logs build-and-deploy-pipeline -L
```

You should delete a PipelineRun when you no longer have a need to reference its logs.  Deleting the PipelineRun deletes the pods that were used
to run the pipeline tasks.

## Check it out in OpenShift Console

OpenShift provides a nice UI for the pipelines and the applications deployed. From the OpenShift console, click **developer** in the upper left drop-down to get to the developer view. Then click **Topology** to view your running app.

![OpenShift Topology](../images/openshift-topology.png)

Click **Pipelines** to explore the pipline your created and explore the PipelineRuns

![OpenShift Pipelines](../images/openshift-pipelines.png)

## Summary

Tekton provides simple, easy-to-learn features for constructing CI/CD pipelines that run on Kubernetes. This tutorial covered the basics to get you started building your own pipelines. There are more features available and many more planned for upcoming releases.

