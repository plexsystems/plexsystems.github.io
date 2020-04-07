---
title: "Deploying Infrastructure on Azure with Atlantis and Kubernetes"
date: "2020-04-05"
author: "John Reese"
github: jpreese
categories: ["azure", "infrastructure", "kubernetes", "terraform"]
featuredImage: "/images/atlantis.jpg"
---

At Plex, our initial adoption of [Terraform](https://www.terraform.io/) was relatively painless. There weren't many teams writing infrastructure as code, and most of the changes that were being deployed through Terraform were new pieces of infrastructure that didn't have any dependencies.

As the number of teams using Terraform and managing their own infrastructure grew, it became apparent that we needed to start putting some processes in place for a few reasons.

## Collaboration was difficult

In order for an engineer to verify their Terraform changes, they needed to be able to run `terraform plan` against live infrastructure. In most cases, this was done locally on their computer. While this approach made is easy for the engineer to get feedback, it presented other problems.

First, in order to successfully run a plan against the existing infrastructure, the engineer themselves needed to have read access to all of the resources that they were making changes to. While this seemed reasonable, it became a problem of scale. Anyone who wanted to be an effective contributor to the code base needed read permissions in Azure. This quickly spiraled out of control and made managing access more difficult.

Secondly, at Plex we require multiple approvers for every pull request. This meant that not only did the engineer creating the pull request have run a `terraform plan` against their changes, but the reviewer of the code as well. This typically meant that the result of a plan was pasted into the pull request so that everyone could see how each resource would be impacted.

## Plans went stale

Unfortunately, infrastructure can change at a moment's notice. Especially when multiple contributors are working on the same infrastructure.

Consider the following scenario:

An engineer opens a pull request against a Terraform module and views the resulting plan. This plan highlights which resources will be created, destroyed, and/or updated. The plan looks good to them, but they decide to hold off on deploying the change because it's a Friday. Give that engineer a gold star!

Monday morning rolls around, and another engineer opens a pull request against the same module. Nothing seems out of the ordinary in the plan, so they apply the plan to production.

Now unfortunately for the first engineer in our scenario, their plan is no longer current.

At this point the original plan has lost most, if not all value. Worst yet, if the original developer is unaware of the newly introduced change, an apply to production could cause problems. Lots... of problems.

## The homegrown solution

To address these problems, we wrote two PowerShell scripts (one for plan and one for apply) that would be executed on our build agents in Azure. These were wired up as policy checks in the repository, allowing us to run them on demand or automatically with webhooks.

When a pull request was created, the plan script would execute `terraform plan` against the changes and output the results of the plan to the pull request comment thread. This enabled reviewers to see the actual changes that were being introduced, without having to run the plan locally themselves. Awesome! Our first problem was solved.

Then, if the plan looked good and the appropriate reviewers approved the change, another script would handle the `terraform apply` step.

To solve the issue of stale plans, we added a policy that in order for a `terraform apply` to be executed, an associated plan needed to be run at least 30 minutes prior. It wasn't ideal, but shortened the window of failure and gave us enough confidence that the original plan was still valid.

This approach worked well for us in the beginning, but as the amount of managed infrastructure, and number of teams using the solution grew, we started noticing gaps.

Rather than continuing to invest in our homegrown solution, we set out to find an alternative. If the title of the blog didn't already give it away, that solution —was Atlantis.

## Atlantis

[Atlantis](https://www.runatlantis.io/) is a pull request automation tool that makes it easier for teams to manage and deploy their infrastructure.

Conveniently, it addresses the problems that we have already discussed:

- Visual `plan` and `apply` results in the pull request? _Check._
- State locking to guarantee that plans stay relevant? _Check._

As an added bonus, not only does it run on Kubernetes, it's an [open source project](https://github.com/runatlantis/atlantis)!

### How does it work?

Atlantis is controlled by typing commands as comments on the pull request comment thread. Need to run `plan` against your pull request? Just comment on the pull request, `atlantis plan`. This will instruct Atlantis to get the plan for your changes and respond to the pull request with the result.

In order to get the plan, Atlantis will:

1. Clone the repository to its data drive. If you're running Atlantis on Kubernetes, this will either be a local drive within the container (using a Deployment) or a Persistent Volume Claim (using a StatefulSet).

1. Lock the folder containing the changes and store the lock metadata to its data drive. This prevents plans from getting stale.

1. Execute `terraform plan` against the folder containing the change.

1. Report the plan result as a comment in the pull request.

This results in the following workflow:

{{< figure src="/images/atlantis_workflow.png" >}}

With this approach, engineers no longer need to have access to the managed infrastructure. All plan and apply operations are handled within the pull request.

As an added bonus, Atlantis also supports [Web Hooks](https://docs.microsoft.com/en-us/azure/devops/service-hooks/services/webhooks?view=azure-devops), which enable actions to be performed automatically, such as when a pull request is opened or the code within the pull request is modified.

## Rolling out Atlantis

One of the first decision points during our roll out of Atlantis was: _How are we going to host this?_. Given that Atlantis runs inside of a Docker container, we had a lot of options in front of us. Should we stand up an [Azure Container Instance](https://docs.microsoft.com/en-us/azure/container-instances/)? Kubernetes? _Spolier alert, it's probably Kubernetes._

Regardless of the final home for Atlantis, there are some preliminary steps that we had to complete.

### 1. Create the Atlantis user (optional)

Atlantis will need to use a user account that will be responsible for cloning the repositories and responding to pull requests. We decided to create a user named `atlantis`. _Completely original, we know._

We also had to create a Personal Access Token (PAT) for this user. At a minimum it needs to have the following scopes:

- `Code (Read & Write)`
- `Code (Status)`

```text
NOTE: It is possible to use any account assuming it has the proper permissions.
```

### 2. Customize the Atlantis image

Atlantis provides a base image that enables basic terraform workflows, but we quickly found that it did not satisfy our needs.

Plex [air gaps](https://en.wikipedia.org/wiki/Air_gap_(networking)) its pipelines, so it wouldn't be possible to download Terraform plugins on demand. Additionally, we leverage [Terragrunt](https://github.com/gruntwork-io/terragrunt) to help keep our infrastructure codebases DRY. 

Neither of these concerns are addressed in the base image, so we wrote our own custom `Dockerfile` that was based off of the original Atlantis image:

{{< gist jpreese f538d049246cb73639dafffa07153355 >}}

Our `Dockerfile` not only contains Atlantis, but:

- Terragrunt for DRY infrastructure code.
- The Azure CLI to be able to provision and manage Azure resources.
- [Terraform Bundle](https://github.com/hashicorp/terraform/tree/master/tools/terraform-bundle) to explicitly configure which plugins are pre-installed.
- .. and a small helper script to assist with authorization to Azure DevOps.

While Atlantis _does_ have a `--write-git-creds` flag which will write out a `.git-credentials` file and configure the `git` client within the Docker image. We found that at the time of our implementation (and writing this post), it always assumed an `ssh` connection.

In other words, if, like us, you reference your Terraform modules via `https`:

```hcl
module "foo" {
  source = "git::https://org@dev.azure.com/org/project/_git/modules//foo?ref=0.1.0"
}
```

Atlantis won't be able to pull your private module repositories within Azure DevOps.

To work around this, we implemented a [git credential helper](https://git-scm.com/docs/gitcredentials) that sets the git username and password to the credentials that are already passed into the environment.

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

While the above Dockerfile may be more than you need, at a minimum, you'll most likely need the Azure CLI if you intend on managing Azure resources.

### 3. Find a home for Atlantis

A lot of our workloads at Plex already run on Kubernetes, so it seemed like the most reasonable choice for Atlantis as well.

Better yet, the Atlantis team already provides the Kubernetes manifests, as well as examples on how to configure them. Both a Deployment and StatefulSet example are included [here](https://www.runatlantis.io/docs/deployment.html#statefulset-manifest). We ended up choosing the StatefulSet approach, but your requirements may be different.

The biggest factor to consider when choosing between the two approaches is that a Deployment won't persist the locks on your repositories if the pod is restarted, whereas a StatefulSet will.

Atlantis stores all of its data (e.g. locks, plugins) in its `data directory`. When using a Deployment, this will just be a folder within the container, which has no guarentees to persist if the pod is restarted. On the other hand, a StatefulSet will use a Persistent Volume Claim, allowing the locks to persist, even through a restart.

Regardless of your choice, all that needs to be done is to find the `Azure DevOps Config` section and replace the placeholder values with the credentials that were created in the previous step.

### 4. Configure the webhooks

At this point, we had:

- A user to clone repositories and comment on pull requests with the proper access.
- A Dockerfile with the tools needed to complete our workflows.
- The Atlantis manifests applied to our Kubernetes cluster.

The last major step in getting up and running with Atlantis, was to add the webhooks that would respond to events within our repository (e.g. pull request created, pull request updated).

Similar to the Kubernetes manifests, Atlantis provides great [documentation](https://www.runatlantis.io/docs/configuring-webhooks.html#azure-devops) on how to configure these for Azure DevOps.

We just followed the steps, tested the connection, and began using Atlantis to review and deploy our infrastructure changes!

## Returning to the surface

Our journey to Atlantis was long, but we intend to vacation here for quite awhile.

Teams can now create custom infrastructure workflows that make sense for them, and we now have a central authority for all infrastructure changes. Better yet, it allowed us to retire our legacy workflows and like many others, adopt an open source solution with a great community behind it.

We would like to give a special thanks to [@mcdafydd](https://twitter.com/mcdafydd) for adding the Azure DevOps to Atlantis. As we continue to use Atlantis, we hope to contribute back to the project and make managing infrastructure as code with Atlantis, even better.
