---
title: "Deploying Infrastructure on Azure with Atlantis and Kubernetes"
date: "2020-04-05"
author: "John Reese"
github: jpreese
categories: ["azure", "infrastructure", "kubernetes", "terraform"]
featuredImage: "/images/atlantis.jpg"
---

The initial rollout of [Terraform](https://www.terraform.io/) at Plex, 

Our initial adoption of Terraform was relatively painless. There weren't many teams using writing infrastructure as code, and most of the changes that were being deployed through Terraform were new pieces of infrastructure.

However, as we increased the amount of resources managed by Terraform, it became apparent that we needed to start putting some processes in place.

## Collaboration was difficult

In order to verify the infrastructure changes, engineers needed to run `terraform plan`. Typically, this was done locally on their computer, and while this approach made is easy for the engineer to feedback, it presented other problems.

First, in order to successfully run a plan against the existing infrastructure, the engineer themselves needed to have read access to all of the resources that they would be making changes to. If even just a service principal. While this may seem reasonable, it was more of a problem of scale. Anyone who wanted to be an effective contributor to the code base would need permissions in Azure to do so, which can quickly spiral out of control and makes managing access more difficult.

Secondly, we require multiple approvers for every pull request at Plex. This meant that not only the contributor had to run a plan against their changes, but the reviewer as well. Or even worse, just look at changes that were made to the Terraform configurations. This typically meant that the result of a plan was pasted into the pull request so that everyone could clearly see how each resource would be impacted.

## Plans went stale

Unfortunately, infrastructure can change at a moment's notice. Especially when multiple contributors working on the same infrastructure.

Consider the following scenario:

An engineer opens a pull request against a Terraform module and views the resulting plan.

This plan highlights which resources will be created, destroyed, and/or updated. The plan looks good to them, but they decide to hold off on deploying the change because it's a Friday. Give that engineer a gold star!

Monday morning rolls around, and another engineer opens a pull request against the same module. Nothing seems out of the ordinary in the plan, so they apply the plan to production.

Now unfortunately for the first engineer in our scenario, their plan is no longer current.

At this point the plan has lost most, if not all value. Worst yet, if the original developer is unaware of the newly introduced change, an apply to production could cause problems. Lots... of problems.

## The homegrown solution

To address these problems, we wrote a couple PowerShell scripts (one for plan and one for apply) that would be executed on our build agents in Azure. These were wired up as policy checks in the repository, allowing us to run them on demand or automatically with webhooks.

When a pull request was created, the plan script would execute `terraform plan` against the changes and output the results of the plan to the pull request comment thread within Azure DevOps. This enabled reviewers to see the actual changes that were being introduced, without having to run the plan locally themselves. Awesome! Our first problem was solved.

Then, if the plan looked good and the appropriate reviewers approved the change, another script would handle the `terraform apply`.

To solve the issue of stale plans, we added a policy that in order for an apply to be executed, an associated plan needed to be run at least 30 minutes prior. It wasn't ideal, but shortened the window of failure and gave us enough confidence that the result of the plan was still valid.

This approach worked well for us in the beginning, but as the amount of managed infrastructure, and number of teams using the solution grew, we started noticing gaps.

Rather than continuing to invest in our homegrown solution, we set out to find an alternative. If the title of the blog didn't already give it away, that solution —was Atlantis.

## Atlantis

[Atlantis](https://www.runatlantis.io/) is a pull request automation tool that makes it easier for teams to deploy their infrastructure changes.

Conveniently, it addresses the problems that we have already discussed:

- Visual `plan` and `apply` results in the pull request? _Check._
- State locking to guarantee that plans stay relevant? _Check._

As an added bonus, not only does it run on Kubernetes, it's an [open source project](https://github.com/runatlantis/atlantis).

### How does it work?

Atlantis is controlled by typing commands as comments on the pull request comment thread. Need to run `plan` against your pull request? Just comment on the pull request, `atlantis plan`. This will instruct Atlantis to get the plan for your changes and respond to the pull request with the result.

To get the plan for you, Atlantis will:

1. Clone the repository to its data drive. If you're running Atlantis on Kubernetes, this will either be a local drive within the container (using a Deployment) or a Persistent Volume Claim (using a StatefulSet).

1. Lock the folder containing the changes and store the lock metadata to its data drive. This prevents plans from getting stale.

1. Execute `terraform plan` against the folder containing the change.

1. Report the plan result as a comment in the pull request.

This results in the following workflow:

{{< figure src="/images/atlantis_workflow.png" >}}

With this approach, engineers no longer need to have access to the managed infrastructure. All plan and apply operations are handled within the pull request.

As an added bonus, Atlantis also supports [Web Hooks](https://docs.microsoft.com/en-us/azure/devops/service-hooks/services/webhooks?view=azure-devops), which enable actions to be performed automatically, such as when a pull request is opened or the code within the pull request is modified.

## Rolling out Atlantis

At Plex, a lot of our workloads already run on Kubernetes, and we felt Atlantis should be no different.

The first step in deploying Atlantis is to add some additional resources to your version control system. In our case, this was Azure DevOps.

### Create an Atlantis user (optional)

Atlantis will need to use a user account that will be responsible for cloning the repositories and responding to pull requests. We decided to create a user named `atlantis`. I know, it's pretty clever.

Additionally, you will need to create a Personal Access Token (PAT) for this user. At a minimum it should have the following scopes:

- `Code (Read & Write)`
- `Code (Status)`

```text
NOTE: It is possible to use any account assuming it has the proper permissions.
```

### Customize your Atlantis image

Atlantis provides a base image that enables basic terraform workflows, but it's possible to create you, like us, have additional requirements.

Plex [air gaps](https://en.wikipedia.org/wiki/Air_gap_(networking)) its pipelines, so it wouldn't be possible to download Terraform plugins on demand. Additionally, we leverage [Terragrunt](https://github.com/gruntwork-io/terragrunt) to help keep our infrastructure codebases DRY.

To address these issues, we wrote our own custom `Dockerfile` that is based off of the original Atlantis image:

{{< gist jpreese f538d049246cb73639dafffa07153355 >}}

This `Dockerfile` contains not only Atlantis, but:

- Terragrunt for DRY infrastructure code.
- The Azure CLI to be able to provision and manage Azure resources.
- [Terraform Bundle](https://github.com/hashicorp/terraform/tree/master/tools/terraform-bundle) to explicitly configure which plugins are allowed.
- .. and a small helper script to assist with authorization to Azure DevOps.

Atlantis _does_ have a `--write-git-creds` flag which will write out a `.git-credentials` file and configure the `git` client within the Docker image with the credentials that were passed in.

However, at the time of our implementation (and writing this post), it always assumes an `ssh` connection.

If your Terraform modules are referenced via `https`, e.g.:
```hcl
module "foo" {
  source = "git::https://org@dev.azure.com/org/project/_git/modules//foo?ref=0.1.0"
}
```

One approach to be able to successfully authorization to your Azure DevOps is to add a [git credential helper](https://git-scm.com/docs/gitcredentials), similar to the following:

_.gitconfig_
```sh
[credential "https://dev.azure.com"]
	helper = "/bin/sh /home/atlantis/azure-devops-helper.sh"
```

_azure-devops-helper.sh_
```sh
#!/bin/sh

# These values are already provided to the container from
# the Kubernetes manifest.
echo username=$ATLANTIS_AZUREDEVOPS_WEBHOOK_USER
echo password=$ATLANTIS_AZUREDEVOPS_TOKEN
```

At a minimum, if you intend to manage Azure resources, the Azure CLI will need to be added to the Dockerfile!

### Apply the manifests to your cluster

The Atlantis team already provides a good example on how to configure your Kubernetes manifests. Both a Deployment and StatefulSet example are included [here](https://www.runatlantis.io/docs/deployment.html#statefulset-manifest).

```text
NOTE: The StatefulSet manifests adds an ATLANTIS_DATA_DIR variable.
Do not include this if using a Deployment!
```

Look for the section titled `Azure DevOps` config and replace the placeholder values with the user credential values that were obtained in the previous step.

If you prefer a slower rollout of Atlantis and only want to manage a few, or even a single repository, the `ATLANTIS_REPO_WHITELIST` variable instructs Atlantis to only watch certain repositories.

### Setting up the webhooks

At this point, we should have:

- A user to clone repositories and comment on pull requests with the proper access.
- A Dockerfile with the tools needed to complete your workflows.
- The Atlantis manifests applied to your cluster of choice.

The last major step in getting up and running with Atlantis, is to add the web hooks to respond to events within your repository (e.g. pull request created, pull request updated).

Similar to the Kubernetes manifests, Atlantis provides great [documentation](https://www.runatlantis.io/docs/configuring-webhooks.html#azure-devops) on how to configure these for Azure DevOps.

Once these are applied, you are ready to rock and roll!

## Returning to the surface

Our journey to Atlantis was long, but we intend to vacation here for quite awhile.

Teams can now create custom infrastructure workflows that make sense for them, and we now have a central authority for all infrastructure changes. Better yet, it allowed us to get rid of a custom solution and embrace open source.

We would like to give a special thanks to [@mcdafydd](https://twitter.com/mcdafydd) for adding the Azure DevOps to Atlantis. As we continue to use Atlantis, we hope to contribute back to the project and make managing infrastructure as code even easier.
