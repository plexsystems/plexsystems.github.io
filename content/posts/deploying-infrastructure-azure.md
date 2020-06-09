---
title: "Deploying Atlantis for Azure DevOps onto Kubernetes"
date: "2020-04-05"
author: "John Reese"
github: jpreese
categories: ["azure", "infrastructure", "kubernetes", "terraform"]
featuredImage: "/images/atlantis.jpg"
---

At Plex, our initial adoption of [Terraform](https://www.terraform.io/) was relatively painless. There weren't many teams writing infrastructure as code, and most of the changes that were being deployed through Terraform were new pieces of infrastructure that didn't have any dependencies.

As the number of teams using Terraform and managing their own infrastructure grew, it became apparent that we needed to start putting some processes in place for a few reasons.

## Collaboration was difficult

In order for an engineer to verify their Terraform changes, they needed to be able to run `terraform plan` against live infrastructure. In most cases, this was done locally on their computer. While this approach made it easy for the engineer to get feedback, it presented other problems.

First, to successfully run a plan against the existing infrastructure, the engineer needed to have read access to all of the resources that they were making changes to. While this seemed reasonable, it became a problem of scale. Anyone who wanted to be an effective contributor to the code base needed read permissions in Azure. This quickly spiraled out of control and made managing access more difficult.

Secondly, at Plex we require multiple approvers for every pull request. This meant that not only did the engineer creating the pull request have to run a `terraform plan` against their changes, but the reviewer of the code as well. This typically meant that the result of a plan was pasted into the pull request so that everyone could see how each resource would be impacted.

## Problems with different Terraform clients

Not all engineers have the same Terraform client installed on their computer. This can cause problems when working on the same backing Terraform state.

When executing a `terraform refresh` or `terraform apply` against the current state, the version of Terraform that was used is stored in the state file. Attempting to execute Terraform commands with an older Terraform client than what is listed in the state file will result in an error similar to the following:

```text
Terraform doesn't allow running any operations against a state
that was written by a future Terraform version. The state is
reporting it is written by Terraform '0.12.0'
```

It's important to note that newer Terraform clients can change the syntax of the state file and how the results should be interpreted. Unfortunately, older Terraform clients will not know how to process this newer information and will need to be updated.

This means that if an engineer has Terraform `0.12.1` on their machine and executes a `refresh` or `apply`, the state will be updated to include version `0.12.1` and everyone will need to upgrade their clients.

## Plans went stale

Unfortunately, infrastructure can change at a moment's notice. Especially when multiple contributors are working on the same infrastructure. Consider the following scenario:

An engineer opens a pull request against a Terraform module and views the resulting plan. This plan highlights which resources will be created, destroyed, and/or updated. The plan looks good, but they decide to hold off on deploying the change because it's a Friday. Give that engineer a gold star!

Monday morning rolls around, and another engineer opens a pull request against the same module. Nothing seems out of the ordinary in the plan, so they apply the plan to production.

Now unfortunately for the first engineer in our scenario, their plan is no longer current.

At this point the original plan has lost most, if not all value. Worst yet, if the original developer is unaware of the newly introduced change, an apply to production could cause problems. Lots of problems.

## The homegrown solution

To address these problems, we wrote two PowerShell scripts (one for plan and one for apply) that would be executed on our build agents in Azure. These were wired up as policy checks in the repository, allowing us to run them on-demand or automatically with Azure [webhooks](https://docs.microsoft.com/en-us/azure/devops/service-hooks/services/webhooks?view=azure-devops).

When a pull request was created, the plan script would execute `terraform plan` against the changes and output the results of the plan to the pull request comment thread. This enabled reviewers to see the actual changes that were being introduced, without having to run the plan locally themselves. Awesome! Our first problem was solved.

Then, if the plan looked good and the appropriate reviewers approved the change, another script would handle the `terraform apply` step.

To solve the issue of stale plans, we added a policy that for a `terraform apply` to be executed, an associated plan needed to be run at least 30 minutes prior. It wasn't ideal, but shortened the window of failure and gave us enough confidence that the original plan was still valid.

This approach worked well for us in the beginning, but as the amount of managed infrastructure, and the number of teams using the solution grew, we started noticing gaps.

Rather than continuing to invest in our homegrown solution, we set out to find an alternative. If the title of the blog didn't already give it away, that solution was Atlantis.

## Atlantis

[Atlantis](https://www.runatlantis.io/) is a pull request automation tool that makes it easier for teams to manage and deploy their infrastructure.

Conveniently, it addresses the problems that we have already discussed:

- Visual `plan` and `apply` results in the pull request? _Check._
- Each repository can set the required version of Terraform? _Check._
- State locking to guarantee that plans stay relevant? _Check._

Best of all, not only does it run on Kubernetes, it's an [open source project](https://github.com/runatlantis/atlantis)!

### How it works

Atlantis is controlled by typing commands as comments on the pull request comment thread. Need to run `plan` against your pull request? Just comment on the pull request: `atlantis plan`. This will instruct Atlantis to get the plan for your changes and respond to the pull request with the result.

To get the plan, Atlantis will:

1. Clone the repository to its data drive. If you're running Atlantis on Kubernetes, this will either be a local drive within the container (using a Deployment) or a Persistent Volume Claim (using a StatefulSet).

1. Lock the folder containing the changes and store the lock metadata to its data drive. This prevents plans from getting stale and prevents conflicts from multiple contributors.

1. Execute `terraform plan` against the folder containing the change.

1. Report the plan result as a comment in the pull request.

This results in the following workflow:

{{< figure src="/images/atlantis_workflow.png" >}}

With this approach, engineers no longer need to have access to the managed infrastructure. All plan and apply operations are handled within the pull request.

As a bonus, Atlantis also supports webhooks, which enable actions to be performed automatically, such as when a pull request is opened or the code within the pull request is modified.

## Rolling out Atlantis

One of the first decision points during our roll-out of Atlantis was: _How are we going to host this?_. Given that Atlantis runs inside of a Docker container, we had several options in front of us. Should we stand up an [Azure Container Instance](https://docs.microsoft.com/en-us/azure/container-instances/)? Kubernetes? _Spolier alert, it's probably Kubernetes._

Regardless of the final home for Atlantis, there are some preliminary steps that we had to complete.

### 1. Create the Atlantis user

Atlantis will need to use a user account that will be responsible for cloning the repositories and responding to pull requests. We decided to create a user named `atlantis`. _Completely original, I know._

We also created a Personal Access Token (PAT) for this user with the minimum scope possible:

- `Code (Read & Write)`
- `Code (Status)`

### 2. Configure Atlantis for Terragrunt

We use [Terragrunt](https://github.com/gruntwork-io/terragrunt) in our infrastructure repositories to help keep our infrastructure codebases DRY. Unfortunately, Atlantis does not support Terragrunt workflows out of the box and only provides a default workflow which excutes basic Terraform commands.

However, Atlantis allows you to create your own workflows if you provide a [server side repository configuration](https://www.runatlantis.io/docs/server-side-repo-config.html). This enabled us to extend Atlantis and run `terragrunt` commands.

```yaml
repos:
# The list of repositories Atlantis watches. Supports wildcards/*
- id: dev.azure.com/org/project/first-repo

  # Only allow "apply" comments if the PR is approved and can be merged
  apply_requirements: [approved, mergeable]
  workflow: terragrunt

# Instead of terraform commands, run these for plan and apply
workflows:
  terragrunt:
    plan:
      steps:
      - run: terragrunt init -input=false -no-color
      - run: terragrunt plan -input=false -no-color -out $PLANFILE
    apply:
      steps:
      - run: terragrunt apply -input=false -no-color $PLANFILE
```

```subtext
repository-config.yaml
```

### 3. Customize the Atlantis image

Plex [air gaps](https://en.wikipedia.org/wiki/Air_gap_(networking)) its infrastructure pipelines, so it's not possible to download plugins on demand. Instead, we leverage Terraform's plugin cache and download the plugins ahead of time.

To accomplish this, we wrote our own custom `Dockerfile` that was based off of the original Atlantis image:

```Dockerfile
FROM golang:1.13 as builder
RUN apt-get update \
  && apt-get install unzip

# Install terraform-bundle
RUN git clone \
  --depth 1 \
  --single-branch \
  --branch "v0.12.0" \
  https://github.com/hashicorp/terraform.git \
  $GOPATH/src/github.com/hashicorp/terraform
RUN cd $GOPATH/src/github.com/hashicorp/terraform \
  && go install ./tools/terraform-bundle

# Download plugins
COPY terraform-bundle.hcl .
RUN terraform-bundle package -os=linux -arch=amd64 terraform-bundle.hcl
RUN mkdir /go/tmp \
  && unzip /go/terraform_*-bundle*_linux_amd64.zip -d /go/tmp

FROM runatlantis/atlantis:v0.11.1
ENV TF_IN_AUTOMATION="true"
ENV TF_CLI_ARGS_init="-plugin-dir=/home/atlantis/.atlantis/plugin-cache"

# Install Azure CLI
ARG AZURE_CLI_VERSION="2.0.74"
RUN apk add py-pip \
  && apk add --virtual=build gcc libffi-dev musl-dev openssl-dev python-dev make
RUN pip --no-cache-dir install azure-cli==${AZURE_CLI_VERSION}

# Install Terragrunt
ARG TERRAGRUNT_VERSION="v0.23.2"
RUN curl -L -o /usr/local/bin/terragrunt https://github.com/gruntwork-io/terragrunt/releases/download/${TERRAGRUNT_VERSION}/terragrunt_linux_amd64 \
  && chmod +x /usr/local/bin/terragrunt
  
# Copy plugins
COPY .terraformrc /root/.terraformrc
COPY --chown=atlantis:atlantis --from=builder /go/tmp /home/atlantis/.atlantis/plugin-cache
RUN mv /home/atlantis/.atlantis/plugin-cache/terraform /usr/local/bin/terraform

# Configure git
COPY .gitconfig /home/atlantis/.gitconfig
COPY azure-devops-helper.sh /home/atlantis/azure-devops-helper.sh

# Copy server-side repository config
COPY repository-config.yaml /home/atlantis/repository-config.yaml

CMD ["server", "--repo-config=/home/atlantis/repository-config.yaml"]

LABEL org.opencontainers.image.title="Atlantis Environment"
LABEL org.opencontainers.image.description="An environment to support executing Terragrunt operations with Atlantis"

LABEL binary.atlantis.version="v0.11.1"
LABEL binary.terragrunt.version=${TERRAGRUNT_VERSION}
LABEL binary.azure-cli.version=${AZURE_CLI_VERSION}
```

```subtext
dockerfile
```

Our `Dockerfile` not only contains Atlantis, but:

- Terragrunt for DRY infrastructure code.
- The Azure CLI to be able to provision and manage Azure resources.
- [Terraform Bundle](https://github.com/hashicorp/terraform/tree/master/tools/terraform-bundle) to explicitly configure which plugins are pre-installed.

```hcl
# The version of Terraform to include with the bundle.
terraform {
  version = "0.12.21"
}

# The providers to pre-download and include in the Atlantis image.
providers {
  azurerm     = ["~> 2.0.0"]
  azuread     = ["~> 0.7.0"]
  random      = ["~> 2.2.0"]
  local       = ["~> 1.4.0"]
}
```

```subtext
terraform-bundle.hcl
```

.. as well as small helper script to assist with authorization to Azure DevOps.

While Atlantis _does_ have a `--write-git-creds` flag which will write out a `.git-credentials` file and configure the `git` client within the Docker image. We found that at the time of our implementation (and writing this post), it always assumed an `ssh` connection.

In other words if, like us, you reference your Terraform modules via `https`:

```hcl
module "foo" {
  source = "git::https://org@dev.azure.com/org/project/_git/modules//foo?ref=0.1.0"
}
```

Atlantis won't be able to pull your private module repositories within Azure DevOps.

To work around this, we implemented a [git credential helper](https://git-scm.com/docs/gitcredentials) that sets the git username and password to the credentials that have already passed into the environment.

```bash
[credential "https://dev.azure.com"]
	helper = "/bin/sh /home/atlantis/azure-devops-helper.sh"
```

```subtext
.gitconfig
```
<br />
```bash
#!/bin/sh 

# These values are already provided to the container from
# the Kubernetes manifest.
echo username=$ATLANTIS_AZUREDEVOPS_WEBHOOK_USER
echo password=$ATLANTIS_AZUREDEVOPS_TOKEN
```

```subtext
azure-devops-helper.sh
```

While the above `Dockerfile` may be more than you need, at a minimum you'll need the Azure CLI if you intend on managing Azure resources.

### 4. Apply the manifests to Kubernetes

Many of our workloads at Plex already run on Kubernetes, so it seemed like the most reasonable choice for Atlantis as well. Better yet, the Atlantis team already provides the Kubernetes manifests, as well as examples on how to configure them. Both a Deployment and StatefulSet example are included [here](https://www.runatlantis.io/docs/deployment.html#statefulset-manifest).

We ended up choosing the StatefulSet approach, but your requirements may be different. The biggest factor to consider when choosing between the two approaches is that a Deployment won't persist the locks on your repositories if the pod is restarted, whereas a StatefulSet will. This is because Atlantis stores all of its data (e.g. locks, plugins) in its `data directory`. When using a Deployment resource, this will just be a folder within the container, which has no guarantees to persist if the pod is restarted. On the other hand, a StatefulSet will use a Persistent Volume Claim which allowis the locks to persist, even through a restart.

Regardless of your choice, all that needs to be done is to find the `Azure DevOps Config` section in the provided example manifests and replace the placeholder values with the credentials that were created in the previous step.

### 5. Configure the webhooks

At this point, we had:

- A user to clone repositories and comment on pull requests with the proper access.
- A Dockerfile with the tools needed to complete our workflows.
- The Atlantis manifests applied to our Kubernetes cluster.

The last major step in getting up and running with Atlantis, was to add the webhooks that would respond to events within our repository.

Similar to the Kubernetes manifests, Atlantis provides great [documentation](https://www.runatlantis.io/docs/configuring-webhooks.html#azure-devops) on how to configure these for Azure DevOps.

We just followed the steps, tested the connection, and began using Atlantis to review and deploy our infrastructure changes!

## Returning to the surface

Our journey to Atlantis was long, but we intend to vacation here for quite a while.

Teams can now create custom infrastructure workflows that make sense for them, and we now have a central authority for all infrastructure changes. Better yet, it allowed us to retire our legacy workflows, and adopt an open source solution with a great community behind it.

We would like to give a special thanks to [@mcdafydd](https://twitter.com/mcdafydd) for adding the Azure DevOps integration to Atlantis. As we continue to use Atlantis, we hope to contribute back to the project and make managing infrastructure with Atlantis even better.
