---
title: "Introducing Sinker: A tool to sync container images from one registry to another"
date: "2020-07-19"
author: "John Reese"
github: jpreese
categories: ["containers", "kubernetes", "open source"]
featuredImage: "/images/plexgithub.png"
---

At Plex, all container images that are used in our environments are sourced from our internal container registries. While this gives us greater control over which images can and cannot be used, it poses a bigger problem. How can we leverage container images that are managed in the public cloud?

A common solution to this problem is to sync the public container image, to your organizations private registry. First pulling down the image, and then pushing it to the internal registry. However, this can be tedious for each and every public image that you intend to use. Furthermore, it's not always obvious where the image was originally sourced from so updating to a newer version could result in a scavanger hunt to track down the source registry!

## Introducing Sinker

Our solution to these problems, as well as others around container image management, was [Sinker](https://github.com/plexsystems/sinker). Sinker is an open source tool that not only pushes public images to an internal registry, but also keeps a manifest of the images that are being used in each repository.

Consider there scenario where your team wants to be able to deploy the [prometheus-operator](https://github.com/coreos/prometheus-operator). To do so, three images are required: the operator itself, the config reloader for the operator, and a configmap reloader.

Each of these images are not only tagged individually, but are also sourced from different registries. Without any sort of record of what image is in use and where the image came from, the upgrade story becomes difficult.

### The Image Manifest

The image manifest is a `.yaml` file that is used by Sinker to perform all of its operations such as pulling, and pushing images. The manifest for the promethus operator would look like the following:

```yaml
target:
  host: organization.com
  repository: team
sources:
- repository: coreos/prometheus-operator
  host: quay.io
  tag: v0.40.0
- repository: coreos/prometheus-config-reloader
  host: quay.io
  tag: v0.40.0
- repository: jimmidyson/configmap-reload
  tag: v0.4.0
```

The `sources` section defines all of the container images that should be pushed to the `target` registry. For example, the `prometheus-operator` image would be pushed to `organization.com/team/coreos/prometheus-operator` and tagged with `v0.40.0`.

With the introducing of the image manifest, there is now a defined desired state configuration for the target registry. The manifest could be added to each repository that needs to manage images, or a central repository that defines which images are available to the entire organization.

## Using Sinker at Plex

We implemented Sinker to solve not only the problem of needing to sync public container images to our private registry, but also to automatically detect which images are in use by our Kubernetes clusters.

### The create and update commands

Sinker's `create` command can optionally take a single `.yaml` file or directory that contains Kubernetes resources, and generate an image manifest with all of the container images that were found. This includes images that are present inside of container arguments and some CRDs.

If we were to run the `create` command on the `bundle.yaml` produced by the prometheus-operator project, we would automatically create a familiar looking manifest.

```shell
$ sinker create bundle.yaml --target containers.plex.com/kubernetes
```

```subtext
terminal
```

Result:

```yaml
target:
  host: containers.plex.com
  repository: kubernetes
sources:
- repository: coreos/prometheus-operator
  host: quay.io
  tag: v0.40.0
- repository: coreos/prometheus-config-reloader
  host: quay.io
  tag: v0.40.0
- repository: jimmidyson/configmap-reload
  tag: v0.4.0
```

The `update` command can then be used to update the image manifest when the Kubernetes resources are modified. This enables our engineers to just worry about making changes to their resources, and not have to add, remove, or update version definitions in two places.

This also reduces human error because the `update` command is executed against the same manifests that will be deployed to the cluster. Sinker is able to figure out an image's origin based on the provided target registry and the container's repository.

### The pull and list commands

When it comes to the local testing of Kubernetes deployments, most of our engineering teams leverage [Kind](https://github.com/kubernetes-sigs/kind). In order for Kind to be able to run container images that require authentication, which is the case for preivate registries, the image needs to be loaded into the cluster.

With Sinker, the `pull` command can be used to pull down all of the `target` (or source) images found in the image manifest.

```shell
$ sinker pull target
INFO[0000] Finding images that need to be pulled ...
INFO[0000] Pulling containers.plex.com/coreos/prometheus-operator:v0.40.0
INFO[0012] Pulling containers.plex.com/coreos/coreos/prometheus-config-reloader:v0.40.0
INFO[0019] Pulling containers.plex.com/jimmidyson/configmap-reload:v0.4.0
INFO[0019] All images have been pulled!  
```

```subtext
terminal
```

With the images pulled down onto the local machine, we can then use Sinker's `list` command to output a list of the target images and load them into Kind.

```shell
for i in `sinker list target`; do
  kind load docker-image ${i} --name $CLUSTER_NAME
done
```

```subtext
bash
```

### A pipeline for adding new images

...

## Something about conclusion

...