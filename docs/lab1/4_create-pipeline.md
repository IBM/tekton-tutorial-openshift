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

## Next

Next, go to [Define a Service Account](5_create-service-account.md).
