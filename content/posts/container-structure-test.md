---
title: "Testing Containers with Container Structure Test"
date: "2020-06-08"
authors:
- github: jpreese
  name: "John Reese"
categories: ["containers", "testing"]
featuredImage: "/images/container-structure-test/featured.jpg"
---

It is no secret that when we are writing software, tests are a critical component to ensure the code _actually_ does what we say it does. It is so critical that most languages come with testing frameworks. JavaScript has testing frameworks such as [mocha](https://mochajs.org/) and [jasmine](https://jasmine.github.io/). Go ships with its own testing capabilities provided by the [testing package](https://golang.org/pkg/testing/). And while writing tests in these languages is an accepted standard practice, all too often we forget that there is more to getting an application onto production than the app itself.

[Dockerfiles](https://docs.docker.com/engine/reference/builder/) play a big part in how we ship software at Plex. We use them heavily in our build pipelines to run [containerized jobs](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/container-phases?view=azure-devops) as well as the distribution mechanism to get the software onto our Kubernetes clusters.  We like to say, "If it's code, we can test it," and that is no exception when it comes to writing `Dockerfiles`.

## A traditional approach to writing Dockerfiles

When tasked with writing a `Dockerfile`, there are defined requirements that the `Dockerfile` needs to satisfy. Binaries may be required to run linting tools, environment variables need to be set, and files have to exist at specific paths. Consider the `Dockerfile` that was shown in an earlier blog post, [Deploying Atlantis for Azure DevOps onto Kubernetes](../deploying-infrastructure-azure/):

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
RUN curl -L https://github.com/gruntwork-io/terragrunt/releases/download/${TERRAGRUNT_VERSION}/terragrunt_linux_amd64 \
  && -o /usr/local/bin/terragrunt \
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

This `Dockerfile` requires:

- Terraform plugins to support different types of infrastructure.
- The Terragrunt CLI.
- Two environment variables, `TF_IN_AUTOMATION` and `TF_CLI_ARGS_init`.
- Git configuration that points to an Azure DevOps helper script (and the script itself).
- A global configuration for repositories that use Atlantis.

To begin writing this `Dockerfile`, knowing that we'll need to install [terraform-bundle](https://github.com/hashicorp/terraform/tree/master/tools/terraform-bundle) (using Go), as well as install the `unzip` package to unzip the generated bundle, we could jump right into it and create a `Dockerfile`:

```Dockerfile
FROM golang:1.13 as builder
RUN apt-get update \
  && apt-get install unzip
```

```subtext
dockerfile
```

To verify that the base image exists and that the unzip package was successfully installed, we execute the `docker run` command with an interactive terminal to explore the contents of the produced container.

```shell
$ docker build . -t testing:latest
$ docker run -it testing:latest
root@fb06cc45835c:/go# unzip
UnZip 6.00 of 20 April 2009, by Debian. Original by Info-ZIP.
```

```subtext
terminal
```

We know the `unzip` package successfully installed because after executing the binary, we get a response back that includes the version of the binary and the date it was built.

If the `unzip` package did not install successfully, the container would throw an error stating that `unzip` is an unknown command.

```shell
$ docker run -it testing:latest
root@fb06cc45835c:/go# unzip
bash: unzip: command not found
```

```subtext
terminal
```

This cycle of adding commands into the `Dockerfile`, building the image, running the container, and exploring the structure of the container would need to continue until we met all of the requirements. While this process will allow us to satisfy all of the requirements, much of the verification is manual. Besides, when the requirements change, as they tend to do, it can be incredibly challenging to have confidence that we didn't inadvertently break anything else by introducing a change.

## A better way forward with Container Structure Test

[Container Structure Test](https://github.com/GoogleContainerTools/container-structure-test) is a tool developed by Google that enables us to test the structure of a container. I'm not sure how they came up with the name, but what's important is that it removes the need to test `Dockerfiles` manually!

Container Structure Test supports four different types of tests: `command`, `file existence`, `file content`, and `metadata`. Each type of test can validate a different aspect of a container. It does this by running the container for you, automatically, and testing whether or not the structure of the container meets the defined requirements in the test file.

*NOTE: If you would like a deeper dive into the internals of Container Structure Test, the folks over at Google put together a [blog post](https://opensource.googleblog.com/2018/01/container-structure-tests-unit-tests.html) that explains the different types of tests in greater detail, as well as how to get the most out of the tool.*

## Writing a Dockerfile

When equipped with Container Structure Test, rather than creating a new `Dockerfile`, adding commands, building the container, running the container, and manually exploring the container—we define the requirements upfront.

In the previous example, the first known requirement was that the `unzip` package had to be present inside of the container. To define this requirement as a test case within Container Structure Test, we use a`command` test:

```yaml
schemaVersion: 2.0.0

commandTests:
- name: unzip
  command: unzip
  args: ["-v"]
```

```subtext
unzip-test.yaml
```

To test the `Dockerfile`, pass the name of the built image along with the location of the test file to the `container-structure-test` CLI. 

*NOTE: It's possible to save some keystrokes, as well as time, by combining the `docker build` and `container-structure-test test` commands so that every time the `Dockerfile` changes, a new image is built and the test suite is run against the most recently built image.*

```shell
$ docker build . -t testing:latest \
  && container-structure-test test --image testing:latest --config unzip-test.yaml

===================================
============= RESULTS =============
===================================
Passes:      1
Failures:    0
Duration:    443.548204ms
Total tests: 1
```

```subtext
terminal
```

While it's possible to create a single test and then pass the test inside of your `Dockerfile` (which is reminiscent of Test-Driven Development), another valid approach is to define _all_ your requirements first and build the `Dockerfile` from the test suite. As either is acceptable, whichever method makes the most sense for your style is the one you should use!

To complete our example of writing a test suite for the Atlantis container image, this is the completed test suite.

```yaml
schemaVersion: 2.0.0

# Validate the environment contains the required tooling
commandTests:
- name: Atlantis
  command: atlantis
  args: ["version"]

- name: Terraform
  command: terraform
  args: ["version"]
  expectedOutput: ["Terraform v0.12.24"]

- name: Terragrunt
  command: terragrunt
  args: ["--version"]

- name: Azure CLI
  command: az
  args: ["--version"]

# Validate the required configuration files exist
fileExistenceTests:
- name: Server Configuration
  path: /home/atlantis/repository-config.yaml
  shouldExist: true

- name: Azure DevOps Helper
  path: /home/atlantis/azure-devops-helper.sh
  shouldExist: true

- name: Terraform Plugin Cache
  path: /home/atlantis/.atlantis/plugin-cache
  shouldExist: true

# Validate checkpoint functionality is off
fileContentTests:
- name: Terraform Checkpoint Disabled 
  path: /home/atlantis/.terraformrc
  expectedContents: ['disable_checkpoint = true']

# Validate container environment is configured as expected
metadataTest:
  env:
    - key: TF_IN_AUTOMATION
      value: true
    - key: TF_CLI_ARGS_init
      value: -plugin-dir=/home/atlantis/.atlantis/plugin-cache
```

```subtext
atlantis-tests.yaml
```

## Integrating Container Structure Test into your workflow

While using Container Structure Test for local development can save a lot of time when first writing the `Dockerfile`, the benefits don't stop there!

After creating the `Dockerfile`, we now have defined a set of requirements that the `Dockerfile` must adhere to. Having this set of requirements enables us to be able to run Container Structure Test at any point in the future and verify, automatically, that no regressions or unexpected changes occurred inside of the container. A software component that exists inside of a container image could change its version at any time, or even worse, be removed altogether. If you depend on specific components, and particular versions, add tests for them!

In the case of Atlantis, the maintainers are free to [change the version of Terraform](https://github.com/runatlantis/atlantis/commit/a9873aeb585c9e03e89c188feb9dd0e7086ebdf3) inside of the container at any time during their release cycle. Updating from Atlantis `0.12` to `0.13` should be trivial, but because the version of Terraform used to deploy infrastructure [needs to be taken into consideration](https://github.com/hashicorp/terraform/issues/19290#issuecomment-436012086)—we need to be made aware when the version changes.

```yaml
schemaVersion: 2.0.0

- name: Terraform
  command: terraform
  args: ["version"]
  expectedOutput: ["Terraform v0.12.21"]
```

```subtext
terraform-version-test.yaml
```

This `command test` will fail the test suite when the version of Terraform inside of the container changes from `v0.12.21`. Now, upgrading Atlantis is as simple as bumping the version and re-running the test suite.

Integrating Container Structure Test into your workflow for developing `Dockerfiles` automates much of the manual exploration that was previously required to verify the structure of the container. As a bonus, after the `Dockerfile` is first built, we have a test suite that we can run every time we need to make a change to the `Dockerfile` inside of our pipelines.
