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
      value: "1.0"
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
```

This should set the `imageUrl` to `us.icr.io/tekton101lab/picalc`. Run the following `sed` commands,

```bash
sed -i "s/<REGISTRY>/$REGISTRY_ROUTE/g" tekton/run/picalc-pipeline-run.yaml
sed -i "s/<NAMESPACE>/$NAMESPACE/g" tekton/run/picalc-pipeline-run.yaml

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

## Next

Next, go to [Run the Pipeline](7_run-the-pipeline.md).
