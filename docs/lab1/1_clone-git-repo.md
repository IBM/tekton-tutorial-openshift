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

## Next

Next, go to [Create a Task to Build and Upload Container Image to Registry using Kaniko](2_build-and-push-image-using-kaniko.md).
