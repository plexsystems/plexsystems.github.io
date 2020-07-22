---
title: "Introducing Sinker: A tool to sync container images from one registry to another"
date: "2020-07-19"
authors:
- github: jpreese
  name: "John Reese"
categories: ["containers", "kubernetes", "open source"]
featuredImage: "/images/introducing-sinker/featured.png"
---

At Plex, all container images in our environments are sourced from our internal container registries. While this gives us greater control over which images can and cannot be used in our environments, it poses a bigger problem. How can we leverage container images that are managed in the public cloud?

A common solution to this problem is to sync the public container image to your organization's private registry by pulling down the image and then pushing it to your internal registry. However, this can be tedious for each public image that you intend to use. Furthermore, the origin of the original image is not always obvious, so updating to a newer version could result in a scavenger hunt to track down the original registry.

Our solution to these problems, as well as others around container image management, was to build [Sinker](https://github.com/plexsystems/sinker).

## Introducing Sinker

Sinker is an open-source tool that not only pushes public images to an internal registry but also keeps a manifest of the images that are being used in each repository.

Consider the scenario where your team wants to be able to deploy the [prometheus-operator](https://github.com/coreos/prometheus-operator). To deploy the operator, three images are required: the operator itself, the config reloader for the operator, and a ConfigMap reloader.

Each of these images are not only tagged individually but are also sourced from different registries. Without any record of what image is in use and where the image came from, the upgrade story becomes difficult.

### The Image Manifest

The image manifest is a `.yaml` file that is used by Sinker to perform all of its operations such as pulling and pushing images. The manifest for the prometheus operator would look like the following:

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

```subtext
.images.yaml - an example image manifest
```

The `sources` section defines all of the container images that should be pushed to the `target` registry. For example, the `prometheus-operator` image would be pushed to `organization.com/team/coreos/prometheus-operator` and tagged with `v0.40.0`.

With the introduction of the image manifest, there is now a defined desired state configuration for the target registry. The manifest could be added to each repository that needs to manage images or a central repository that defines which images are available to the entire organization.

## Using Sinker at Plex

We implemented Sinker to solve not only the problem of syncing public container images to our private registry but also to automatically detect which images are in use by our Kubernetes clusters.

Sinker's `create` command can optionally take a single `.yaml` file or directory that contains Kubernetes resources, and generate an image manifest with all of the found container images. Container images that are not only in the `image` field, but also images that are present inside of container arguments and some CRDs.

If we were to run the `create` command on the `bundle.yaml` produced by the prometheus-operator project, we would automatically create a familiar-looking manifest.

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

```subtext
.images.yaml
```

Using the `update` command will then update the image manifest when the Kubernetes resources are modified. Automatically updating the manifest allows our engineers to only worry about making changes to their resources, and not adding, removing, or updating version definitions in two places. Automatic manifest updates also reduces human error because the `update` command is executed against the same manifests that will be deployed to the cluster.

When it comes to the local testing of Kubernetes deployments, most of our engineering teams leverage [Kind](https://github.com/kubernetes-sigs/kind). For Kind to be able to run container images that require authentication (which is usually the case for private registries), the image needs to be pre-loaded into the cluster.

To load all of the images into Kind, we can use Sinker's `pull` command. The `pull` command downloads all of the `target` images found in the image manifest onto the host machine.

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

```bash
for i in `sinker list target`; do
  kind load docker-image ${i} --name $CLUSTER_NAME
done
```

```subtext
bash
```

### Our process for adding and updating new images

One of the best use cases we've found for Sinker is to incorporate it into a dedicated pipeline for updating and adding new images to our internal container registry. Moving the process of container image management into a pipeline ensures that every image present in the internal registry has been accepted for use.

When an engineer needs to add or update a container image, the pull request should include the image manifest. During the review process, it's easy to see which image the engineer wants to add to the registry. If the source host is untrusted, or if it's a beta release, there might need to be a broader discussion during the review process.

If the pull request is approved, the pipeline will first pull down all of the images present in the manifest. Immediately after pulling the images, we run a security vulnerability scan against all of the images.

```bash
while read -r dockerImage;
do
  twistcli images scan $dockerImage
  VIOLATION=$?
  if [ $VIOLATION != 0 ]; then
    FOUND_VIOLATION=1
  fi
done < `sinker list source`
exit $FOUND_VIOLATION
```

```subtext
bash
```

We list the _source_ images here because we want to validate whether or not the image has any known vulnerabilities that exceed our comfort level _before_ they are sync'd and approved for use in our internal registry. If the image exceeds our threshold, the pipeline will fail the build, and the image will not be added to the internal registry.

After the vulnerability scan is complete and passes all of our thresholds, we then [lock](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-image-lock) all of the newly pushed images to ensure that they cannot be overwritten or deleted.

```bash
while read -r dockerImage;
do
  az acr repository update --name containers --image $dockerImage \
    --write-enabled false \
    --delete-enabled false
done < `sinker list target`
```

```subtext
bash
```

This pipeline guarantees that every image that our engineers use has already been pre-scanned for vulnerabilities. Additionally, there is no risk of the image being accidentally deleted or overwritten, thanks to Azure's image locking mechanisms.

## What's next

Sinker has solved several problems at Plex, but these problems are not unique to us! We open-sourced Sinker to help other organizations solve similar problems when working with syncing container images. If you're interested in trying Sinker for yourself, it can be downloaded from our [releases](https://github.com/plexsystems/sinker/releases) page on GitHub.

We have some features in mind for future releases but are always looking for additional use cases to support the broader community. If you have a use case that isn't supported yet, add an issue to the repository. We'd love to hear from you!
