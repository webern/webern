# Twoliter

## Out of Tree Builds for Bottlerocket (OOTB)

This document summarizes my work at AWS as a senior engineer leading the Out-of-Tree Builds (OOTB)
initiative for Bottlerocket.
The goal was to allow customers to create and build their own Bottlerocket variants without forking
the entire Bottlerocket monorepo using a new build tool named Twoliter.

## Introduction to Bottlerocket

Bottlerocket is a Linux distribution made by AWS,
purpose built for running container orchestrators (like Rancher OS or Container OS).

### Features:

* Immutable/read-only root filesystem
* DM/Verity
* SELinux
* Secure Boot
* Tempfs/ephemeral for some root filesystem directories such as /etc
* The system will not boot if the root filesystem is altered.
  Manual configuration changes will not persist.
* Settings API
    * Configuration,
      e.g.
      motd,
      DNS,
      k8s static pods are set through:
        * API Domain Socket
        * SE Linux protected
        * Settings are stored in a custom data store on the persistent/user-data partition
        * Also/typically,
          set by instance userdata during the boot (TOML)
    * User configuration settings are used to generate /etc files through a templating system.
    * EXAMPLE:
        * apiclient set motd="Hello Bottlerocket",
        * sends a POST to the api socket,
          gets a 201
        * the new value is in the datastore
        * The system knows /etc/motd depends on that setting and the file is regenerated with the
          new value (also generated on boot since it's on a tempfs
        * Partition layout: A/B for updates,
          immutable.
          User Data for containers/settings/persistance

### Customers

A LOT of machines are running Bottlerocket!

1. EKS: some significant and growing percentage of EKS nodes are running Bottlerocket.
   (link): goal to become the default.
2. ECS: tiny user base
3. New! Fargate: Platform version 1.4 is built on Bottlerocket (link).
4. New! EKS Auto Mode uses Bottlerocket (link).

### Variants

A key value proposition with Bottlerocket is that SOFTWARE CANNOT BE INSTALLED.

The minimal software necessary to run the orchestrator is installed at image build time and cannot
be altered later.
It's the whole point!

Bottlerocket launched with support for:

* EKS
* ECS

This requires the build system to install different software at image creation time.
These differences are called variants.

Additionally,
each different version (esp.
w/K8s) requires a different variant.
So Bottlerocket has variants with names like:

* aws-k8s-1.20
* aws-k8s-1.21
* aws-k8s-1.22
* aws-ecs-1
* aws-ecs-2

## Bottlerocket Build System 101

Before my time (I started a year after the first line of code was written): Bottlerocket considered
systems like Buildroot and Yocto for creating Linux images,
and decided to build their own system.

Some considerations:

* Bottlerocket wanted to use Rust for 1st-Party Code and wanted to provide a consistent Rust-based
  developer experience.
* The cargo (Rust) build system provides incredible flexibility.
* Consistent builds of all software using an "SDK" with all the right toolchains.

Bottlerocket decided to drive builds with Cargo.
It's cool!

### How the build Works (Used to Work)

I wrote an AWS engineering blog about this: How the Bottlerocket Build System Works

#### RPM and Packages

* The best way to build a Linux image is with a package manager to install everything into a mounted
  image.
  Bottlerocket uses RPM/DNF.
* There is no package manager installed with Bottlerocket,
  it is only used at image creation time.
* Packages: Builds of software like the Kernel,
  Systemd,
  Kubernetes,
  etc.,
  are driven by RPM.spec files and RPM build commands inside the Bottlerocket SDK container.

#### Buildsys and other custom tools

Before the Bottlerocket build would start,
certain,
custom command line tools,
written in Rust,
were built with the local version of Rust.

* buildsys: responsible for kicking off builds in the Bottlerocket SDK container environment.
  E.g.
  Buildsys is going to build things like the Kernel and Systemd inside a container called the
  Bottlerocket SDK.
  The builds are driven by this Dockerfile.
* pubsys: for pushing build artifacts (images and update repositories),
  managing keys,
  etc.
* testsys: used to kick off tests using the bottlerocket-test-system (I designed and built this
  system as well,
  BTW!)

#### Hacking with Cargo

Bottlerocket needed a way to model dependencies between packages,
for example:

* Before binutils can be built,
  glibc and libz need to be built.
* This is represented in RPM spec,
  meaning that the rpm build command knows that it needs to install glibc.rpm,
  and libz.rpm before it can build binutils.
* But how do we know that we need to build glibc.rpm and libz.rpm before we build rpm build
  binutils?

You could parse RPM.spec and create your own build graph… Or… Hack cargo!

In short,
we used Cargo's build orchestration to manage RPM dependency builds using Rust-native tooling.

#### Description of the Cargo Hacking

Each RPM.spec package is modeled as if it were a Cargo (Rust) project (a "crate").
Cargo,
running on the host,
orchestrates a build just as if it were building Rust libraries.

But each "library" is a tiny little empty binary.
The only purpose of the build was to run a build.rs script.

Rust allows you to run arbitrary code before building a Rust project by creating a build.rs file.
This is often used to generate Rust code or compile C code or do whatever is needed before the
build.
In our case we use it to invoke Buildsys,
which launches a docker build command (using buildx) that results in RPM files being output to the
build directory!

#### What this Means

But the main takeaway is the build system expected to be sitting in the same tree with all the
packages it was building,
and it expected all built packages to be in the build directory before creating a Bottlerocket
image.

## The Problem: Custom Variants

The Bottlerocket build system relied on all package definitions being in the same tree.
The Build system resided in the same tree with all the package definitions.
Variant definitions were also in-tree with all of Bottlerocket's sourcecode.

This meant that,
to create a custom variant,
a user would have to:

* Fork the Bottlerocket monorepo
* Introduce Packages and their Variant definition on top of the Fork
* Maintain very painful breaking rebases

Users were effectively prevented from maintaining their own custom variants,
though a few brave early adopters did so,
it was not a sustainable solution.

## The Requirements

* Users should have their own project-based git repository containing only their customizations of
  Bottlerocket (packages and variants).
  They should not need to fork the sourcode they are not modifying.
* Users should be able to use the Bottlerocket-team provided packages without building everything
  from source themself.
  (For example,
  they shouldn't have to build the kernel if they aren't modifying.
  There are hundreds of packages that they aren't modifying and building them takes 40 minutes.)
* Users should be able to composite their own packages with those provided by the Bottlerocket team.
* Users should be able to publish their Bottlerocket packages for further use downstream in their
  org or publicly.
  (For example Datadog could produce a datadog-agent.rpm package for Bottlerocket and publish it.)
* Users should be able to define their own Bottlerocket settings.
  (This is an entirely parallel arm of the OOTB project with original design by Sean Kelley).
* Don't Break It! The Bottlerocket release cadence cannot stop!

## The Solution: Twoliter

### High level:

* Extract the build system from the main Bottlerocket repository and provide it as its own
  binary-released tooling.
* Publish Bottlerocket packages as composable/reusable entities.
* Define a project structure and project file for composing your own Bottlerocket variant.
* Ingest and use built RPM packages from Kits.

**Twoliter:** A new command line tool for building,
publishing and testing Bottlerocket images and Kits.

**Kits:** Atomically vended groups of related RPM packages to be ingested and used when building
Bottlerocket images.

The Bottlerocket Mono Repo would be broken into 4 separate projects:

* **Bottlerocket-core-kit:** A Twoliter project for vending the majority of the packages needed to
  build a Bottlerocket instance.
* **Bottlerocket-kernel-kit:** A Twoliter project for building the various versions of the Linux
  kernel,
  microcode,
  and
* **Twoliter:** A new binary for interacting with Bottlerocket projects (including all the old build
  system components).
* **Bottlerocket:** The old monorepo became a Twoliter project for definition the K8s and ECS AMIs,
  having had all of the packages and the old build system removed from it,
  it now ingests Bottlerocket-core-kit and Bottlerocket-kernel-kit to make images.

### Kits

Kits and Twoliter projects were the core innovation that would allow users to create their own
variants.
The idea is simple:

* Allow for the vending of Bottlerocket RPM packages.
* Change the build system to allow for the ingestion of these packages instead of building
  everything from local sourcecode.
* Allow customers to define and publish their own kits if they so desire.
  Ideally an ecosystem of kits could grow,
  but this hasn't happened yet.

Though the idea was simple,
changing the very complex build system to support this was a heavy lift with many nuances and
complexities.
I designed it,
initiated the implementation,
and when engineers became available they were added to the project,
which I led.

Parallel Problems and Designs that required additional engineering resources (designed by another
L6,
but I led the L5 engineers that were implementing this during the other L6's leave):

* Removing all conditional compilation
* Make Bottlerocket settings modular and installable

## Project Dynamics

### The Stakes: Fargate

AL2 support was expiring.
Fargate needed a new platform and really wanted it to be on Bottlerocket (not AL 2023).
Bottlerocket could not add these proprietary AMIs to the public git tree.
Bottlerocket could not support building and testing the AMIs themselves.
Fargate could not work with a messy fork of the Bottlerocket git tree.

**Fargate Absolutely Needed Out of Tree Builds**

### The Project Timeline was Aggressive

Many additional features were needed,
not just OOTB.
e.g.
Linux kernel changes,
support for new features in the OS,
XFS,
etc.
etc.

Many of the features needed were easier to understand and easier to prioritize than OOTB.
But OOTB was absolutely critical path along with everything else.

## Project Phases

### Ideation

The principal engineer,
another L6,
and myself discussed and wrote many design proposal docs,
had many brainstorming meetings over the course of what might have actually been nearly two years.
This project was highly ambiguous.

### Design

When we honed in on the kits idea,
I wrote the final design document through several iterations.
It is available here: Twoliter Design Document.

### Build System Extraction

We needed to extract the build system and begin vending Twoliter before we could begin making major
changes to the build system.
We needed to decouple Twoliter development from Bottlerocket development.

**Twoliter make:** A facade over cargo make which was the underlying entrypoint of the old build
system.
Invocations of twoliter make are passed through directly to cargo make along with all relevant
environment variables and command line arguments allowing the Bottlerocket project to be built the
same way that it always,
just now it was being done by passing it's commands into a binary released tool called Twoliter.
This required a hard cut-over and it was done without incident.

### Twoliter Projects

Begin to implement the Twoliter project structure,
introduce Twoliter.toml and begin passing responsibility to the project structure.

### Agile Concession: Twoliter Alpha

Fargate absolutely needed to start building their own images now to facilitate development of their
next platform,
1.4,
and there was no way we were going to have kits ready in time.

We created an "Alpha" version of Twoliter to solve this problem.
We "tricked" the build system into accepting built packages from a dockerfile.

We realized that if we did not include the build-time dependency on these packages,
but included the metadata dependency,
then the build system would not attempt to build them,
but would attempt to install them.

We created some "Alpha" pathways in the code that would pull from a Docker container,
and slay out all the RPM files into the Build directory as if they had just been built,
then we initiated the regular build process and it "just worked."

This gave the Fargate a Kit-like experience,
despite the fact that kits were not implemented yet.

**Note:** I argued against this concession because I was afraid we would add additional customers
and end up in the Alpha stage forever without implementing kits.
I was afraid we would never get the engineering resources we needed to get kits over the line.
The principal engineer had more experience with getting things done at AWS and promised me that we
were implementing kits.

### Local Kits

Deep changes to change dependency management,
RPM output,
RPM install and project structure to allow for the definition and building of kits.
I did the implementation of this.

### External Kits

New features to allow for the ingestion of RPM packages from downloaded Docker images.
Implemented by Jarred.

### Publishing Kits

Changes to the publication system to allow for the publishing of Kits.
Implemented by Sam.

### Bottlerocket Kit Party 🎉

A week-long group effort to break up the Bottlerocket monorepo into Bottlerocket-core-kit and
Bottlerocket (images) using the newly released Twoliter.
Rapid iteration and releases to fix bugs in Twoliter.
Team effort,
all-hands-on-deck,
Arnaldo was hands on keyboard the most for the Bottlerocket repo.
Jarred was hands-on-keyboard the most for the Twoliter repo.

### Revise Fargate Project

Offer assistance to Fargate to switch their project over from Twoliter Alpha to proper Twoliter
Kits.
