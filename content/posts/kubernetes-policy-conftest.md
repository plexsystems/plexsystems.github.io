---
title: "Accelerated Feedback Loops when Developing for Kubernetes with Conftest"
date: "2020-05-23"
author: "John Reese"
github: jpreese
categories: ["kubernetes", "policy", "security", "code reviews"]
featuredImage: "/images/loop.jpg"
---

The feedback loop when deploying to Kubernetes can be quite slow. Not only does the YAML need to be syntactically correct, but we need to ask ourselves:

Is the API version of our resource definition compatible with the version of Kubernetes that it is being deployed to? Kubernetes is constantly evolving, and over time, [deprecates older APIs](https://kubernetes.io/docs/reference/using-api/deprecation-policy/#deprecating-parts-of-the-api) in favor of newer ones. A deployment definition may successfully apply on one version of Kubernetes, but not another.

Are the resources that depend on one another configured properly? For example, when creating a [Service](https://kubernetes.io/docs/concepts/services-networking/service/), you can specify [selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) for which Pods the traffic should ultimately be routed to. In the event that a Service is configured with selectors, but is unable to find any Pods, this mistake would not be known until the Service is deployed to the cluster. What makes this case even more challenging is that this configuration is technically valid but would require additional testing to verify if the traffic is flowing as expected.

[Kind](https://github.com/kubernetes-sigs/kind) aims to help solve a lot of these concerns by allowing developers to spin up a Kubernetes cluster on their local machine, verify their changes, and tear it down with ease. However, it can still take a fair amount of time to bring up a Kind cluster, apply the manifests, and test the outcome.

While these concerns can be caught relatively early, there are additional considerations, especially in the realm of security, that may not be immediately obvious and could go unnoticed until they become a [serious problem](https://unit42.paloaltonetworks.com/non-root-containers-kubernetes-cve-2019-11245-care/).

To catch a lot of these problems ahead of time without the need of a Kubernetes cluster, including Kind, Plex validates all deployments to Kubernetes against policies. A lot of policies.

## An example policy

Most of our policies are written in [Rego](https://www.openpolicyagent.org/docs/latest/policy-language/), a policy language that is interpreted by the [Open Policy Agent](https://www.openpolicyagent.org/) (OPA). To better understand Rego, and how we can leverage it to write policies for Kubernetes, consider the following scenario:

In our cluster, we want to be able to quickly identify which team owns a given Namespace. This can be useful for being able to notify teams about overutilization, cost reporting, problematic pods, and more. To accomplish this, we require that all namespaces must be created with an `owner` label.

To show how we can use Rego to validate Kubernetes manifests, let's create a namespace without an `owner` label:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: missinglabel
```

We'll also need a policy. A Rego policy to enforce this requirement would look like the following:

{{< gist jpreese 42a367e9cc01323931d863bba2777ad1 >}}

*NOTE: The `input` prefix is special when writing policies. It refers to the input document which is one of the base documents provided by the OPA [document model](https://www.openpolicyagent.org/docs/latest/philosophy/#the-opa-document-model).*

This policy defines a single [rule](https://www.openpolicyagent.org/docs/latest/policy-language/#rules) called `violation`. While the order of the statements within the rule do not matter, it can be easier to understand how a rule is evaluated if expressed in this way.

**Line 04** first evaluates if the input has a `kind` property and if its value is equal to Namespace. If the `kind` is not a Namespace, or there does not exist a `kind` property at all, the input will not be considered a violation. The example Namespace has both a `kind` property and has a value of "Namespace", so the statement in the rule would return true.

**Line 05** then checks to see if there exists a key named `owner` in the labels map within the manifests metadata. If there is a key named `owner`, then Namespace must have an owner label. In the example Namespace, there is not an `owner` label so this statement also returns true.

**Line 07** is an assignment operation which will always return true by default.

After all of the statements have been evaluated, the rule itself can be evaluated. In order for a rule to be true, _all_ of the statements inside of the rule must also be true. In this example, all of the statements returned true so the example Namespace would trigger the violation.

## Validating Kubernetes manifests with Conftest

It is important to note that the Open Policy Agent always expects _JSON_ in order to evaluate policies. Kubernetes on the other hand, speaks YAML. At its core, the Open Policy Agent is just a policy engine—it's intended to be generic and fit many use cases.

[Conftest](https://github.com/open-policy-agent/conftest) is a tool that focuses on the user experience when interacting with OPA. Most notably, it handles converting multiple file formats such as `.hcl`, `Dockerfile`, and even `yaml` into JSON so that it can be interpreted by OPA. At Plex, we use Conftest in many of our pipelines. And because Conftest is just a CLI that can be downloaded onto other developer's machines, many of our developers verify their manifests against our policies on their local environments before ever creating a pull request.

Here is an example of a policy that enforces resource constraints on all containers:

{{< gist jpreese b02db173a52fdf2257dee5cab4b33ade >}}

If we were to run Conftest against the [nginx ingress controller](https://github.com/kubernetes/ingress-nginx/blob/master/deploy/static/provider/cloud/deploy.yaml) bundle, we would see that it fails our policy:

```shell
$ conftest test bundle.yaml

FAIL - (ingress-nginx): Container constraints must be specified.
```

We can then take the necessary steps to add the resource constraints into the Deployment so that our CI will allow the bundle onto our cluster.

## Using policy bundles

With Conftest, there's a lot of freedom in the ability to write our own policies, but there are a lot of [bundles](https://www.conftest.dev/sharing/) that the community has written that we also leverage.

A policy bundle can be thought of as a collection of Rego policies that can be pulled from a remote source. A lot of best practices are generic enough that it wouldn't make sense for everyone to have to write and rewrite the same policies. While the concept of bundling and distributing Rego policies for Kubernetes is still quite new, there do exist a couple bundles that have provided immediate value to our pipelines.

### Verify API compatibility with Deprek8ion

[Deprek8ion](https://github.com/swade1987/deprek8ion) is a set of Rego policies that can be used to see if any of our resources are currently, or will be, deprecated in a given Kubernetes release.

```shell
$ conftest pull https://github.com/swade1987/deprek8ion -p policy/deprek8ion
$ conftest test bundle.yaml

WARN - ingress-nginx-admission: API admissionregistration.k8s.io/v1beta1
is deprecated in Kubernetes 1.19, use admissionregistration.k8s.io/v1 instead.
```

### Find security concerns with Kubesec

[Kubesec](https://kubesec.io/) is a set of Rego policies that can be used to see if any of our resources have any insecure configurations.

```shell
$ conftest pull github.com/instrumenta/policies/kubernetes -p policy/kubesec
$ conftest test bundle.yaml

FAIL - Deployment ingress-nginx-controller does not drop all capabilities
FAIL - Deployment ingress-nginx-controller is not using a read only root filesystem
FAIL - Deployment ingress-nginx-controller allows privileged escalation
FAIL - Deployment ingress-nginx-controller is running as root
```

Conftest enables us run policies against multiple resources at once—it is simple, yet powerful. No matter where the Kubernetes manifests originate from, in house or from the open, we can automatically execute our policies to validate that they're compliant with our requirements. This approach allows us to automate our standards and security compliant concerns, freeing up developers to focus on other tasks.

## Investing in a future with policy

The general-purpose approach that the Open Policy Agent has taken, and the user experience that Conftest provides, enables near unlimited use cases for policy-based validation. We've invested heavily in using policy to validate our Kubernetes resources, but already have plans to leverage policy in other scenarios. Notably continuous Kubernetes cluster auditing with [Gatekeeper](https://github.com/open-policy-agent/gatekeeper), and infrastructure security compliance with [Regula](https://github.com/fugue/regula).

Policy-based validation is still relatively new to the Kubernetes ecosystem, but has already made a large impact for us and we're excited to see what's coming next in this space.
