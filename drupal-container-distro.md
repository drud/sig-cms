# Buildpacks For Drupal

Status: Pending

Version: Alpha

Implementation Owner: Drud Technologies

## Motivation

Containers have become the defacto means of distributing and running applications on modern infrastructure. Containers, specifically the [Open Container Initiative(OCI)](https://www.opencontainers.org/) standard, is composed of file system bundles or layers. These layers are often mantained by seperate, unrelated authors and inter-layer compatability can be difficult to determine. Dockerfiles, which are the most popular manifests used by developers to build container images, have some limitations:

1. Image Inter-Layer Compatability: base/dependency image layers are not clearly defined in a consistent manner aside from the image declared in the `FROM` directive. A layer which may perform a `composer install` expects PHP and composer to be installed in a previous layer. Layer compatability needs to be defined in a clear interface in order to enable an ecosystem where layers mantained by different authors can declare compatability requirements using unique ids and versioning.
2. Conditional Layers and Environment Detection: the syntax of Dockerfiles tend to make it difficult to perform conditional steps within a layer. For example, you may not wish to install the `php-apcu` module due to a limitation in the target DrupalSite. This may force you to mantain a seperate Dockerfile for sites with this limitation or include the step of installing this module using shell code to check for the conditions. Layer inclusion should allow for complex detection/condition logic which can be easily defined as code.
3. Patching Upstream Layers: Lets say you have built and deployed a container image `` for your Drupal site which has the following layers:


```
debian:9.8.10 # base OS image. Not mantained directly by the developer
php:7.2.11 # install php and modules. Not mantained directly by the developer
mysite:0.0.1 # add your webroot files to the container files system ie 'ADD . /var/www/html'
```

A security patch is released and push/tagged as `debian:9.8.11` and you would like your site to have this patch. But there is a problem:

```
debian:9.8.11
php:7.2.11 <==== Still references debian:9.8.10
mysite:0.0.1
```

One solution to this problem would be to take the Dockerfiles, if they are available upstream and add their steps to your own Dockerfile. This effectively means you will need to take on the additional burden of mantaining these layers. A solution which defines the lifecycle and contract between layers would need to address this shortcoming

4. Delineate Build and Runtime Layers: It is common for a Dockerfile to install tools, such as `git` to an image layer in order to perform additional tasks such as running a depency management tool like `composer`. In the past, depedencies needed at build-time but not at run-time would either be left in the container image or steps requiring tools such as `git` would be performed outside of the container and copied in. `ADD ...` or `COPY`.
   
   Leaving build related depencies in a web container exposes a larger attack surface for someone to exploit. It also can lead to a overall container size which is 2-10x larger compared to a container image which only has the minumum set of depencies installed. Large container images can negatively impact site deployment and, if it is running on a platform such as Kubernetes, lengthen the time it takes for a site to self heal due to the overhead of `docker pull`
   
   [Multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/), have enabled image authors to avoid most of these problems, but consider the following. What if layers authored by different individuals require the same dependency? They would both need provide their own builder containers which repetetively perform the same installation pre-requisites, leading to longer build times.

5. Ease of Use: In order to build a container locally, you must use a Dockerfile located on your local filesystem. There are certainly was of distruting the Dockerfile or "recipe" for building a site using tools such as Github, but there is no clear registry to find a Dockerfile which would be approriate for building your particular site. This problem is has been resolved with well known staples such as rpm, dockerhub and symphony. What if a user only needs to have Docker(or Kaniko) installed locally along with one other CLI tool which can inspect the target site filesystem, recommend a build recipe for the site based on its observations and perform a build for the user?


## Proposal

Create a set of container builders and standard practices using [Clound Native Buildpacks(v3)](https://buildpacks.io/docs/). Buildpacks are now a incubation project officially part of the Cloud Native Computing Foundation which originated from the CloudFoundry community. 

**It is important to note that this document refers to [buildpacks specification version 3](https://github.com/buildpack/spec) and details and terminology should be confused with earlier versions used by CloudFoundry or Heroku.**

### Terminology

Terms used by Cloud Native Buildpacks and how they apply to us.

#### Buildpacks

A Buildpack represents a building block of executable programs or scripts, identified using a `buildpack.toml`, used to auto-detect the characteristics of an application. One to many Buildpacks compose the [build lifecycle](https://github.com/buildpack/lifecycle#lifecycle) are used by a builder to generate the image layers.

![buildpack diagram](https://buildpacks.io/docs/using-pack/build.svg)
[Reference](https://buildpacks.io/docs/using-pack/building-app/)

Buildpacks can declaratively indicate which Stacks they are compatible with in their configuration. A buildpack should be designed as a self contained step which performs just one task. `Install PHP` and `Composer Install` should be two seperate buildpacks instead of being implemented as a single pack. This allows a seperate Builders to optionally use `Install PHP` and `Composer Install` buildpacks.

Each buildpack is given an opportunity to perform discovery logic in the `detect` portion of the build lifecycle. `Composer Install` `detect` executes logic to discover if the site being built needs to install composer in the build image and executing a `composer install` a build time if it sees a `composer.json` in the project root. If the buildpack does not report it has action to take, then its `build` executable will not be invoked. This means a builder including a composer buildpack could be used to build DrupalSites regardless of composer being used.

This pattern can be used by organizations or hosting providers wishing to add their own custom buildpacks and stacks while leveraging upstream, community mantained buildpacks.

#### Builders

A Builder is an special container image which includes a collection of Buildpacks which would be executed against an application/site, in the order defined by the Builder’s configuration file. Buildpacks are executed not at the time when the Builder is created( `pack create-builder` ), but when the Builder is used to execute a `pack run`. They are identified with a unique stack id which can be used by other Builders and Stacks.

Builders can be distributed via a container registry and the correct builder for an application can be "suggested" by the `pack cli`

#### Stacks

A stack represents a buildpack lifecycle, which a builder executes in order to create a runtime image for the site. A stack can be identified using a unique stack id.

An example stack for Drupal 8, mantained by the drupal community:
```
== Operating System Layer ==
== PHP Runtime ==
== Drupal Installation From Distribution ==
== Custom Codebase ==
```

#### Build Image

A build image is a base image in a stack that buildpacks use to install dependencies which are needed in order to perform perform the build tasks needed for the run image. Example: The composer buildpack, assuming detect=true, will install the composer binaries as a layer in the BUILD container and it will use that binary to perform the `composer install` in the RUN container.

#### Run Image

Is the image, composed of a base image and layers added by buildpacks in order to provide a runnable container image of your application.

## Use Cases

TODO. Will need a proof of concept to demo these.

### Build a Site Locally

TODO. use `pack` with the Drupal builder to build a site container in a single command

### Perform Security Updates

TODO. Use `pack rebase` to rebuild an image based on upstream changes.

### Create a Distribution(stack)

TODO. Show how mantainers of a distribution of Drupal will use buildpacks and how that builder can declare dependencies

### Perform a new site installation

TODO. This would show using ddev to do a new site install and get a dump of the DB.
