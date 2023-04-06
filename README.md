> **Note**: 
> Original content from [How to Build Ansible Execution Environments with OpenShift Pipelines](https://cloud.redhat.com/blog/how-to-build-ansible-execution-environments-with-openshift-pipelines).

# ansible-ee-gitops

First create the project namespace

```bash
oc new-project <your-namespace>
```

## Private Registry Secret

Then create secrets manually for your private registries (pull and push images) and for the Tekton Trigger webhook.
For example, for your private registries. In this case we am pushing to `quay.io` and also pulling from `registry.redhat.io`.

```yaml
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
    -n <your-namespace> \
    --docker-server=<your-registry-server> \
    --docker-username=<your-name> \ 
    --docker-password=<your-password>
```

Then you need to link the secret `<secret-name>` to the `pipeline` SA so it can be used for pulling and pushing images.
```bash
oc secret link pipeline <secret-name> --for=pull,mount -n <your-namespace>
```

> **Warning**: 
> If registry login differ (e.g. `registry.redhat.io` and `private.automation.hub.io`),
> you will need to create and link both secrets to `pipeline` serviceaccount.

## Install pipeline

Clone Pipeline manifests repo
```bash
git clone https://github.com/jayissi/ansible-ee-gitops.git && cd ansible-ee-gitops
```

Edit the Trigger Template `listener/4-trigger-template.yaml` and set the PipelineRun `NAME` parameter to set the image repository name.
```yaml
- apiVersion: tekton.dev/v1beta1
  kind: PipelineRun
  metadata:
    annotations:
    labels:
      tekton.dev/pipeline: ansible-builder
    generateName: ansible-ee-triggered-run-
  spec:
    params:
    - name: ANSIBLE_BUILDER_IMAGE
      value: >-
        registry.redhat.io/ansible-automation-platform-23/ansible-builder-rhel8:latest
    - name: NAME
      value: XXXXXXXXXX
```

Apply change to your cluster
```bash
oc -n <your-namespace> apply -f listener/
```

## Webhook Secret Token

And for the webhook secret token:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ansible-ee-trigger-secret
type: Generic
stringData:
  secretToken: "123"
```
