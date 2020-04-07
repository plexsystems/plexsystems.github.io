---
title: "Deploying Infrastucture on Azure with Atlantis and Kubernetes"
date: "2020-04-05"
author: "John Reese"
github: jpreese
categories: ["infrastructure", "terraform", "kubernetes"]
featuredImage: "/images/atlantis.jpg"
---

The initial rollout of [Terraform](https://www.terraform.io/) at Plex, 

Our initial adoption of Terraform was relatively painless. There weren't many teams using writing infrastructure as code, and most of the changes that were being deployed through Terraform were new pieces of infrastructure.

However, as we increased the amount of resources managed by Terraform, it became apparent that we needed to start putting some processes in place.

## Collaboration was difficult

In order to verify the infrastructure changes, engineers needed to run `terraform plan`. Typically this was done locally on their computer, and while this approach made is easy for the engineer to feedback, it presented other problems.

First, in order to successfully run a plan against the existing infrastructure, the engineer themselves needed to have read access to all of the resources that they would be making changes to. If even just a service principal. While this may seem reasonable, it was more of a problem of scale. Anyone who wanted to be an effective contributor to the code base would need permissions in Azure to do so, which can quickly spiral out of control and makes managing access more difficult.

Secondly, we require multiple approvers for every pull request at Plex. This meant that not only the contributor had to run a plan against their changes, but the reviewer as well. Or even worse, just look at changes that were made to the Terraform configurations. This typically meant that the result of a plan was pasted into the pull request so that everyone could clearly see how each resource would be impacted.

## Plans went stale

Unfortunately, infrastructure can change at a moments notice. Especially when multiple contributors and teams are working in the same space. A plan that was run, even just five minutes ago, could be obsolete and giving reviewers the wrong information.

## The homegrown solution

To address these problems, we wrote a couple PowerShell scripts (one for `plan` and one for `apply`) that would be executed on our build agents in Azure. These were wired up as policy checks in the repository, allowing us to run them on demand, or automatically with webhooks.

When a pull request was created, the plan script would execute `terraform plan` against the changes and output the results of the plan to the pull request comment thread within Azure DevOps. This enabled reviewers to see the actual changes that were being introduced, without having to run the plan locally themselves. Awesome! Our first problem was solved.

Then, if the plan looked good and the appropriate reviewers approved the change, another script would handle the `terraform apply`.

To solve the issue of stale plans, we added a policy that in order for an apply to be executed, an associated plan needed to be run at least 30 minutes prior. It wasn't ideal, but shortend the window of failure and gave us enough confidence that the result of the plan was still valid.

This approach worked well for us in the beginning, but as the amount of managed infrastructure, and number of teams using the solution grew, we started noticing gaps.

Rather than continuing to invest in our homegrown solution, we set out to find an alternative. If the title of the blog post didn't already give it away, that solution was Atlantis.

## What is Atlantis?

[Atlantis](https://www.runatlantis.io/) is a pull request automation tool that makes it easier for teams to deploy their infrastructure changes.

Conveniently, it addresses the problems that we have already discussed:

- Visual `plan` and `apply` results in the pull request? _Check._
- State locking to guarentee that plans stay relevant? _Check._

And best of all? It's [free and open source](https://github.com/runatlantis/atlantis).

### How does it work?

Atlantis is controlled by typing commands as comments on the pull request comment thread. Need to run `plan` against your pull request? Comment on the pull request: `atlantis plan`. This will instruct Atlantis to get the plan for your changes and respond to the pull request with the result.

To get the plan for you, Atlantis will:

1. Clone the repository to its data drive. If you're running Atlantis on Kubernetes, this will either be a local drive within the container (using a Deployment) or a Persistent Volume Claim (using a StatefulSet).

1. Lock the folder containing the changes and store the lock metadata to its data drive. This prevents plans from getting stale.

1. Execute `terraform plan` against the folder containing the change.

1. Report the plan result as a comment in the pull request.

{{< figure src="atlantis_workflow.png" caption="source: " attr="http://medium.com/runatlantis/introducing-atlantis" attrlink="https://medium.com/runatlantis/introducing-atlantis-6570d6de7281">}}

But it gets better! Atlantis also supports [Web Hooks](https://docs.microsoft.com/en-us/azure/devops/service-hooks/services/webhooks?view=azure-devops), which enable actions to be performed automatically such as when a pull request is opened.

///locking
///Master must stay close to production. Drifting master too far ahead of what is deployed to production increases risk.///

---why?//usage
- Plan current changes. Output what will be changed.

- If it looks good, apply.
- Apply restricted based on Azure DevOps policy

- Plan locking based on state
- Keep state small (locks, corruptions)
- Lock freed on PR merge or manual unlock

---setting up
- Public IP, Ingress, Network Security Groups

nsg: out: subnet to public IP
nsg: out: Storage for backend
nsg: in: AzureCloud (20.36.242.132) to Ingress

-- custom dockerfile
--- terragrunt, plugins, airgap (terraform-bundle)
-- enforce version for plan/apply (state problems)

-- custom workflow
-- approved/mergable/terragrunt commands

-- testing
-- container-structure-test to validate configurations and plugins

--closing
special thanks to mcdaffy for adding azdo support
