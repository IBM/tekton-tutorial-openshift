
<!-- Let's use the `tkn` CLI to check the status of the PipelineRun.
While you can check the status of the pipeline using the `kubectl describe` command, the `tkn` cli provides much nicer output.

```
$ tkn pipelinerun describe picalc-pr-c7hsb
Name:              picalc-pr-c7hsb
Namespace:         default
Pipeline Ref:      build-and-deploy-pipeline
Service Account:   pipeline-account

Status

STARTED         DURATION   STATUS
2 minutes ago   ---        Running

Resources

 No resources

Params

 NAME             VALUE
 gitUrl           https://github.com/IBM/tekton-tutorial
 pathToYamlFile   kubernetes/picalc.yaml
 imageUrl         us.icr.io/gregd/picalc
 imageTag         1.0

Taskruns

 NAME                                    TASK NAME         STARTED          DURATION   STATUS
 picalc-pr-c7hsb-build-and-push-image-s8rrg   build-and-push-image   56 seconds ago   ---        Running
 picalc-pr-c7hsb-clone-repo-pvbsk        clone-repo        2 minutes ago    1 minute   Succeeded
```

This tells us that the pipeline is running.
The `clone-repo` pipeline task has completely successfully and the `build-and-push-image` pipeline task is currently running.

Continue to rerun the command to check the status.  If the pipeline runs successfully, the description eventually should look like this:

```
$ tkn pipelinerun describe picalc-pr-c7hsb
Name:              picalc-pr-c7hsb
Namespace:         default
Pipeline Ref:      build-and-deploy-pipeline
Service Account:   pipeline-account

Status

STARTED          DURATION    STATUS
12 minutes ago   2 minutes   Succeeded

Resources

 No resources

Params

 NAME             VALUE
 gitUrl           https://github.com/IBM/tekton-tutorial
 pathToYamlFile   kubernetes/picalc.yaml
 imageUrl         us.icr.io/gregd/picalc
 imageTag         1.0

Taskruns

 NAME                                      TASK NAME           STARTED          DURATION     STATUS
 picalc-pr-c7hsb-deploy-to-cluster-mwvfs   deploy-to-cluster   9 minutes ago    10 seconds   Succeeded
 picalc-pr-c7hsb-build-and-push-image-s8rrg     build-and-push-image     10 minutes ago   1 minute     Succeeded
 picalc-pr-c7hsb-clone-repo-pvbsk          clone-repo          12 minutes ago   1 minute     Succeeded
``` -->
