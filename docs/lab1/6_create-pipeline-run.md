# 6. Create a PipelineRun

You have defined a reusable Pipeline and Task resources for building and deploying an image.
It is now time to look at how you can run the pipeline.

Below is a Tekton PipelineRun resource that defines to run the pipeline we defined above.
You can find this yaml file at [tekton/run/picalc-pipeline-run.yaml](https://github.com/IBM/tekton-tutorial-openshift/blob/master/tekton/run/picalc-pipeline-run.yaml).

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: my-pipelinerun-
spec:
  pipelineRef:
    name: build-and-deploy-pipeline
  params:
    - name: gitUrl
      value: https://github.com/ibm/tekton-tutorial-openshift/
    - name: pathToYamlFile
      value: kubernetes/picalc.yaml
    - name: imageUrl
      value: <REGISTRY>/default/picalc
    - name: imageTag
      value: "1.0"
  serviceAccountName: pipeline-account
  workspaces:
    - name: git-source
      persistentVolumeClaim:
        claimName: picalc-source-pvc
```

Although this file is small there is a lot going on here.  Let's break it down from top to bottom:

* The PipelineRun does not have a fixed name. It uses `generateName` to generate a name each time it is created. This is because a particular PipelineRun resource executes the pipeline only once.  If you want to run the pipeline again, you cannot modify an existing PipelineRun resource to request it to re-run -- you must create a new PipelineRun resource. While you could use `name` to assign a unique name to your PipelineRun each time you create one, it is much easier to use `generateName` and a custom name appended by a hyphen, e.g. `my-pipelinerun-`.

* The Pipeline resource is identified under the `pipelineRef` key.

* Parameters exposed by the pipeline are set to specific values such as the Git repository to clone, the image to build, and the yaml file to deploy. This example builds a [go program that calculates an approximation of pi](https://github.com/IBM/tekton-tutorial-openshift/blob/master/src/picalc.go) called `picalc`. The source includes a [Dockerfile](src/Dockerfile) which runs tests, compiles the code, and builds an image for execution.

  > **NOTE** *You must edit* the `picalc-pipeline-run.yaml` file to substitute the values of `<REGISTRY_ROUTE>` with the information for your private container registry. To find the value for `<REGISTRY_ROUTE>`, enter the command `oc get routes -n openshift-image-registry`.

  ```bash
  export REGISTRY_ROUTE=$(oc get routes -n openshift-image-registry --output json | jq -r '.items[0].spec.host')
  echo $REGISTRY_ROUTE

  vi tekton/run/picalc-pipeline-run.yaml
  ```

* The service account named `pipeline-account` which we created earlier is specified to provide the credentials needed for the pipeline to run successfully.

* The workspace used by the pipeline to clone the Git repository is mapped to a persistent volume claim which is a request for a storage volume.

Before you run the pipeline for the first time, you must create a persistent volume claim for the workspace.

```bash
$ oc create -f tekton/picalc-pipeline-pvc.yaml
persistentvolumeclaim/picalc-source-pvc created
```

The persistent volume claim requests Kubernetes to obtain a storage volume. Since each PipelineRun references the same claim and thus the same volume, this means the PipelineRun can only be run serially to avoid conflicting use of the volume.
There is [funtionality coming in Tekton](https://github.com/tektoncd/pipeline/pull/2326) to allow each PipelineRun to create its own persistent volume claim and thus use its own volume.

Check that the persistent volume claim is bound before continuing.

```bash
$ oc get pvc picalc-source-pvc
NAME                STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS      AGE
picalc-source-pvc   Pending                                      ibmc-block-gold   23s

$ oc get pvc picalc-source-pvc
NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
picalc-source-pvc   Bound    pvc-2ec590d9-13dc-4dbe-b9ee-473ca6e74c91   20Gi       RWO            ibmc-block-gold   78s
```

