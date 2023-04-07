> **Note**: 
> Original content from [How to Build Ansible Execution Environments with OpenShift Pipelines](https://cloud.redhat.com/blog/how-to-build-ansible-execution-environments-with-openshift-pipelines).

>
> For this workshop, you will need CLI access to the **_git_** and **_oc_** or **_kubectl_** command.
>

# ansible-ee-gitops

First set `openshift_project` as a local variable.
```bash
openshift_project=<NAME_OF_OPENSHIFT_PROJECT>
```

Then create the openshift project.
```bash
oc new-project ${openshift_project}
```

## Private Registry Secret

Create secrets manually for your private registries, pull and push images, for your private registries.
In this case we are consuming images from `registry.redhat.io` and pushing to `quay.io`.
```yaml
---
apiVersion: v1
data:
  .dockerconfigjson: XXXXXX
kind: Secret
metadata:
  annotations:
    tekton.dev/docker-0: quay.io
    tekton.dev/docker-1: registry.redhat.io
  name: <secret-name>
  namespace: <your-namespace>
type: kubernetes.io/dockerconfigjson
```
or using CLI

```bash
oc create secret docker-registry <secret-name> \
    -n ${openshift_project} \
    --docker-server=<your-registry-server> \
    --docker-username=<your-name> \
    --docker-password=<your-password>
```

Then you need to link the secret `<secret-name>` to the `pipeline` SA so it can be used for pulling and pushing images.
```bash
oc secret link pipeline <secret-name> \
    -n ${openshift_project} \
    --for=pull,mount
```

> **Warning**: 
> If registry login differ (e.g. `registry.redhat.io` and `private.automation.hub.io`),
> you will need to create and link both secrets to `pipeline` serviceaccount.

## Configure Pipeline

Clone Pipeline manifests repo
```bash
git clone https://github.com/jayissi/ansible-ee-gitops.git && cd ansible-ee-gitops
```

Edit the PipelineRun `pipeline/3-pipeline-run.yaml` and change PipelineRun `git-url` and `NAME` parameters.
```yaml
---
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: ansible-builder-
  labels:
    tekton.dev/pipeline: ansible-builder
spec:
  params:
    - name: git-url
      value: ${{ GIT_CLONE_REPO }}
    - name: NAME
      value: ${{ EE_IMAGE_NAME }}
```

Apply change to your project
```bash
oc -n ${openshift_project} create -f pipeline/
```

## Create Webhook Secret Token

Create the webhook secret token using yaml
```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: ansible-ee-trigger-secret
type: Generic
stringData:
  secretToken: "12345"
```
or CLI

```bash
oc create secret generic ansible-ee-trigger-secret \ 
    -n ${openshift_project} \
    --from-literal=secretToken="12345"
```

## Configure Trigger and EventListener

Install the resources for Trigger, TriggerTemplate, and EventListener
```bash
oc -n ${openshift_project} create -f pipeline/listener/
```

To configure the webhook in Github to point to the URL of our EventListener, we need to get the EventListener route URL.
```bash
oc -n ${openshift_project} get route ansible-ee-el -o jsonpath="http://{.spec.host}{'\n'}"
```

Once complete, we can commit our Execution Environment pipeline to kick off our code.
```bash
git clone https://github.com/<YOUR_FORK>/ansible-execution-environments.git
cd ansible-execution-environments
git commit --allow-empty -m "Empty commit, trigger the pipeline!"
git push origin main
```
