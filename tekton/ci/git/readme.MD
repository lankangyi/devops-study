# Pull source code from git 

1. Create a new Pipeline, pipeline.yaml:
```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: clone-read
```
2. Add the repository URL to the list of Pipeline parameters:
```yaml
spec:
  description: | 
    This pipeline clones a git repo, then echoes the README file to the stdout.
  params:
  - name: repo-url
    type: string
    description: The git repo URL to clone from.
```
3. Add a Workspace, a shared volume to store the code this task is going to download:
```yaml
workspaces:
- name: shared-data
  description: | 
    This workspace contains the cloned repo files, so they can be read by the
    next task.
```

4. Create the Task that is going to use the parameter and the Workplace that you just defined:
```yaml
tasks:
- name: fetch-source
  taskRef:
    name: git-clone
  workspaces:
  - name: output
    workspace: shared-data
  params:
  - name: url
    value: $(params.repo-url)
```

# Create the PipelineRun 
1. Now that you have a Pipeline, to instantiate it and set the actual values, create a PipelineRun, pipelinerun.yaml:
```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: clone-read-run-
spec:
  pipelineRef:
    name: clone-read
  podTemplate:
    securityContext:
      fsGroup: 65532
```
2. Instantiate the Workspace:
```yaml
workspaces:
- name: shared-data
  volumeClaimTemplate:
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
```
3. Set the value of the repository URL parameter:
```yaml
params:
- name: repo-url
  value: https://github.com/tektoncd/website
```

# Git authentication 

1. Create a Kubernetes Secret with your credentials, for example:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-credentials
data:
  id_rsa: AS0tLS...
  known_hosts: AG033S...
  config: GS0FFL...
```
The values for the fields under data are the corresponding base64-encoded files in the .ssh directory. For example, for id_rsa copy-paste the output of:

```shell
cat ~/.ssh/id_rsa | base64 -w0
```
2. Update pipeline.yaml, add a new Workspace to both the Pipeline and the Task:
```yaml
workspaces:
- name: shared-data
  description: | 
    This workspace contains the cloned repo files, so they can be read by the
    next task.
- name: git-credentials
  description: My ssh credentials
tasks:
- name: fetch-source
  taskRef:
    name: git-clone
  workspaces:
  - name: output
    workspace: shared-data
  - name: ssh-directory
    workspace: git-credentials
```
3. Update pipelinerun.yaml to use the Secret as a Workspace and change the git URL from https to SSH:
```shell
workspaces:
- name: shared-data
  volumeClaimTemplate:
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
- name: git-credentials
  secret:
    secretName: git-credentials
params:
- name: repo-url
  value: git@github.com:tektoncd/website.git
```
# Use the source code in a second task 
To learn how to share data between Tasks using a Workspace, you are going to create a second Task that displays the README file from the cloned git repository. You can find more useful examples in the How-to section and the examples folder in the Pipelines git repository.

1. Add a new entry to the tasks section of pipeline.yaml:
```yaml
- name: show-readme
  runAfter: ["fetch-source"]
  taskRef:
    name: show-readme
  workspaces:
  - name: source
    workspace: shared-data
```
2. Create a file called show-readme.yaml and add the following:
```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: show-readme
spec:
  description: Read and display README file.
  workspaces:
  - name: source
  steps:
  - name: read
    image: alpine:latest
    script: | 
      #!/usr/bin/env sh
      cat $(workspaces.source.path)/README.md
```
This Task uses a Workspace that was already set. You don’t have to update pipelinerun.yaml.

# Run the Pipeline 

