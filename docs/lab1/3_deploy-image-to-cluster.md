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
      image: alpine
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

1. The first step runs the `sed` command in an Alpine Linux container and updates the yaml file used for deployment. This step replaces the default image reference `__IMAGE__` by the image that was built by the `kaniko` task `__DIGEST__`. This step requires the yaml file to have two character strings, `__IMAGE__` and `__DIGEST__`, which are substituted with parameter values.

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

## Next

Next, go to [Create a Pipeline](4_create-pipeline.md).
