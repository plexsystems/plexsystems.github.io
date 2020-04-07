---
title: "Deploying Infrastucture on Azure with Atlantis and Kubernetes"
date: "2020-04-05"
author: "John Reese"
github: jpreese
categories: ["infrastructure", "terraform", "kubernetes"]
featuredImage: "/images/atlantis.jpg"
---

At Plex, we manage a lot of our infrastructure using [Terraform](https://www.terraform.io/).

Our initial adoption of Terraform was relatively painless. There weren't many teams using writing infrastructure as code, and most of the changes that were being deployed through Terraform were new pieces of infrastructure.

As we grew, however, it became apparent that we needed to start putting some processes in place. Teams began to work in the same repositories, on the same pieces of infrastructure, and effective reviews were getting increasingly difficult.

## Collaboration was difficult

## Plans went stale

## The homegrown solution

Initially, we implemented our own set of scripts to handle a lot of the pull request orchestration for us.

When a pull request was created, the scripts would run `terraform plan` against the changes and output the results of the plan to the pull request comment thread within Azure DevOps. This enabled reviewers to see the actual changes that were being introduced, without having to run the plan locally themselves.

If the plan looked good, and the appropriate reviewers approved the change, another script would handle the `terraform apply`.

This approach worked well for us in the beginning, but as the amount of managed infrastructure and number of teams using the solution grew, we started noticing gaps.

Take, for example, stale plans. If a contributor in one pull request ran a plan, there were no guarentees that when it was time to apply the changes, that the results of the plan were still valid. Another contributor in a different pull request could have a applied changes, rendering the previous plan obsolete.

We knew these problems were not unique to us, and so we set out to find an alternative and ultimately settled on Atlantis.

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
