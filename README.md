# KubeVela Workflow
[![Go Report Card](https://goreportcard.com/badge/github.com/kubevela/workflow)](https://goreportcard.com/report/github.com/kubevela/workflow)
[![codecov](https://codecov.io/gh/kubevela/workflow/branch/main/graph/badge.svg)](https://codecov.io/gh/kubevela/workflow)
[![LICENSE](https://img.shields.io/github/license/kubevela/workflow.svg?style=flat-square)](/LICENSE)
[![Total alerts](https://img.shields.io/lgtm/alerts/g/kubevela/workflow.svg?logo=lgtm&logoWidth=18)](https://lgtm.com/projects/g/kubevela/workflow/alerts/)

<br/>
<br/>
<h2 align="center">Table of Contents</h2>

* [Why use KubeVela Workflow?](#why-use-kubevela-workflow)
* [How can KubeVela Workflow be used?](#how-can-kubevela-workflow-be-used)
* [Installation](#installation)
* [Quick Start](#quick-start)
* [Features](#features)
* [How to write custom steps?](#how-to-write-custom-steps)
* [Contributing](#contributing)

<br/>
<br/>

<h2 align="center">Why use KubeVela Workflow</h2>

🌬️ **Lightweight Workflow Engine**: KubeVela Workflow won't create a pod or job for process control. Instead, everything can be done in steps and there will be no redundant resource consumption.

✨ **Flexible, Extensible and Programmable**: All the steps are based on the [CUE](https://cuelang.org/) language, which means if you want to customize a new step, you just need to write CUE codes and no need to compile or build anything, KubeVela Workflow will evaluate these codes.

💪 **Rich built-in capabilities**: You can control the process with conditional judgement, inputs/outputs, timeout, etc. You can also use the built-in steps to do some common tasks, such as `deploy resources`, `suspend`, `notification`, `step-group` and more!

🔐 **Safe execution with schema checksum checking**: Every step will be checked with the schema, which means you can't run a step with a wrong parameter. This will ensure the safety of the workflow execution.

<h2 align="center">How can KubeVela Workflow be used</h2>

During the evolution of the [OAM](https://oam.dev/) and [KubeVela project](https://github.com/kubevela/kubevela), **workflow**, as an important part to control the delivery process, has gradually matured. Therefore, we separated the workflow code from the KubeVela repository to make it standalone. As a general workflow engine, it can be used directly or as an SDK by other projects.

### As a standalone workflow engine

Please refer to the [installation](#installation) and [quick start](#quick-start) sections.

### As an SDK

You can use KubeVela Workflow as an SDK to integrate it into your project. For example, the KubeVela Project use it to control the process of application delivery.

You just need to initialize a workflow instance and generate all the task runners with the instance, then execute the task runners. Please check out the example in [Workflow](https://github.com/kubevela/workflow/blob/main/controllers/workflowrun_controller.go#L101) or [KubeVela](https://github.com/kubevela/kubevela/blob/master/pkg/controller/core.oam.dev/v1alpha2/application/application_controller.go#L197).

<h2 align="center">Installation</h2>

### Install Workflow

You can install workflow with Helm:

```shell
helm repo add kubevela https://kubevela.github.io/charts
helm repo update
helm install --create-namespace -n vela-system vela-workflow kubevela/vela-workflow
```

If you have installed KubeVela, you can install Workflow with the KubeVela Addon:

```shell
vela addon enable vela-workflow
```

### Install Vela CLI

Please checkout: [Install Vela CLI](https://kubevela.io/docs/installation/kubernetes#install-vela-cli)

### Install built-in steps in KubeVela(Optional)

Use `vela def apply <directory>` to install built-in steps in [KubeVela](https://github.com/kubevela/kubevela/tree/master/vela-templates/definitions/internal/workflowstep).

Checkout this [doc](https://kubevela.io/docs/end-user/workflow/built-in-workflow-defs) for more details.

<h2 align="center">Quick Start</h2>

You can either run a WorkflowRun directly or from a Workflow Template.

> Note: You need to install the [notification step definition](https://github.com/kubevela/kubevela/blob/master/vela-templates/definitions/internal/workflowstep/notification.cue) and definitions in the [example directory](https://github.com/kubevela/workflow/tree/main/examples/definitions) first to run the example.

### Run a WorkflowRun directly

```yaml
apiVersion: core.oam.dev/v1alpha1
kind: WorkflowRun
metadata:
  name: request-http
  namespace: default
spec:
  workflowSpec:
    steps:
    - name: request
      type: request
      properties:
        url: https://api.github.com/repos/kubevela/workflow
      outputs:
        - name: stars
          valueFrom: |
            import "strconv"
            "Current star count: " + strconv.FormatInt(response["stargazers_count"], 10)
    - name: notification
      type: notification
      inputs:
        - from: stars
          parameterKey: slack.message.text
      properties:
        slack:
          url:
            value: <your slack url>
    - name: failed-notification
      type: notification
      if: status.request.failed
      properties:
        slack:
          url:
            value: <your slack url>
          message:
            text: "Failed to request github"
            
```

Above workflow will send a request to the GitHub API and get the star count of the workflow repository as an output, then use the output as a message to send a notification to your Slack channel.

Apply the WorkflowRun, you can get a message from Slack like:

![slack-success](./static/slack-success.png)

If you change the `url` to an invalid one, you will get a failed notification:

![slack-failed](./static/slack-fail.png)
### Run a WorkflowRun from a Workflow Template

You can also create a Workflow Template and run it with a WorkflowRun with different context.

Workflow Template:

```yaml
apiVersion: core.oam.dev/v1alpha1
kind: Workflow
metadata:
  name: deploy-template
  namespace: default
steps:
  - name: apply
    type: apply-deployment
    if: context.env == "dev"
    properties:
      image: nginx
  - name: apply-test
    type: apply-deployment
    if: context.env == "test"
    properties:
      image: crccheck/hello-world
```

WorkflowRun:

```yaml
apiVersion: core.oam.dev/v1alpha1
kind: WorkflowRun
metadata:
  name: deploy-run
  namespace: default
spec:
  context:
    env: dev
  workflowRef: deploy-template
```

If you apply the WorkflowRun with `dev` in `context.env`, then you'll get a deployment with `nginx` image. If you change the `env` to `test`, you'll get a deployment with `crccheck/hello-world` image.

<h2 align="center">Features</h2>

KubeVela uses Workflow as a SDK to control the process of application delivery. Therefor, all the features of Workflow are also available in KubeVela Workflow.

Please checkout the [KubeVela Workflow documentation](https://kubevela.io/docs/end-user/workflow/overview) for more details.

<h2 align="center">How to write custom steps</h2>

If you're not familiar with CUE, please checkout the [CUE documentation](https://kubevela.io/docs/platform-engineers/cue/basic) first.

You can customize your steps with CUE and some [built-in operations](https://kubevela.io/docs/platform-engineers/workflow/cue-actions). Please checkout the [tutorial](https://kubevela.io/docs/platform-engineers/workflow/workflow) for more details.

<h2 align="center">Contributing</h2>

Check out [CONTRIBUTING](https://kubevela.io/docs/contributor/overview) to see how to develop with KubeVela Workflow.