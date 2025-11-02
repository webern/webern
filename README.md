# Matthew James Briggs

- Experienced systems software engineer
- Working in **Rust** professionally since 2020.

**Call me Matt**

- Q: Why print the full name, including the middle name?
- A: Because there are many people by the name of Matt Briggs in the world.

*Or call me Matthew, or Mateo (I've been working on my Spanish)*

**Webern**

- Q: Why a weird GitHub username?
- A: Because I made it a long time ago before I knew GitHub was important.

Anton Webern was a 12-tone composer of the *Second Viennese School*, buddies with Schönberg and
Berg.
Webern DGAF and wrote tiny, short, sparse pieces,
sometimes so small you can count the notes.
Back when I was in music school, I was impressed and wrote some short, sparse pieces of my own.

**Annoying Disclaimer**

Unless I say otherwise, anything I say on GitHub or anywhere else is my own opinion and not that of
my employers, whether past or present.

---

# Portfolio

This is a guide to some of the PRs, design docs, issues and other artifacts that are useful to find,
especially when I am interviewing for a new position.

## AWS Bottlerocket

I worked on the Bottlerocket team at Amazon from 2020 to 2024 (4.5 years), where I was a systems
engineer working primarily with Rust.

I worked on various projects, including:

- The Rust implementation of The Update Framework (TUF), named [tough].
- The Bottlerocket update system.
- A Kubernasties[^1]-based testing system for Bottlerocket.
- A basic metrics collection system.
- The Twoliter Bottlerocket image customization build tool.

### Twoliter

I led the design and implementation of developer tooling for creating custom variants of
Bottlerocket, called [Twoliter]. When [Fargate] wanted to use Bottlerocket to replace AL2 in its
next platform version, the problem we faced was that Bottlerocket variants were defined in-line in the
Bottlerocket monorepo. The only way for a customer to customize Bottlerocket would be to fork the
monorepo, maintain difficult patches, and build everything from source. We needed to create a
project-based solution for **Out of Tree Builds**.

Twoliter and Kits were the solution. Twoliter is a binary-released, command-line tool for managing
your Bottlerocket variants. It allows you to compose packages provided by the Bottlerocket team in
*Kits* to select what pre-built, Bottlerocket-provided software you want in your image, and allowing
you to add any software of your own to the image.

- [Twoliter Design Document]: I authored this design at the start of the project.
- [Twoliter GitHub Issues]: I initiated the project with a series of GitHub issues that, when
  completed, would realize the design.
- [Twoliter Pull Requests]: My pull requests on the project sorted chronologically.
- [Tokio and Async Stuff]: I often get asked in recruiter screens for Rust jobs if I know how to
  write async code, use `tokio`, etc. Yes, I do.
- [Buildsys Refactoring]: The `buildsys` tool needed a lot of knocking around before it could
  start building and using kits. This was a big refactor to `builder.rs` and consolidation of the
  environment variable interface.
- [CLI Bugfix]: Here's a deceptively simple bugfix of the CLI argument interface. Check out the
  regression testing.
- [Change RPM Output Directory Structure]: It had to change before we could make kits a reality.
- [First Kit Build]: Buildsys builds a toy test kit for the first time!
- [Remove Agile Workaround]: Remove the workaround that we were using to support our first customer
  before kits were finished.

### Signed Migrations

This was an early, Level 5-scoped project in which I altered [tough] and the Bottlerocket update
system such that migration binaries would be verified by cryptographic signature before execution
upon an update reboot.

**Background**

Bottlerocket uses an immutable root filesystem. Most parts of it are read-only, while parts that
need to be writeable (such as `/etc`) are tempfs. In this way, no changes to the root filesystem can
persist a reboot. This is accomplished with secure boot, SELinux and DM-verity.

The system is configured by updating settings through an HTTP service that runs locally on a domain
socket. These user-settings are persisted on the user data partition in a schema called the
*datastore*. On boot, or when settings change, the `/etc` files are generated from templates and
affected services are re-started.

A Bottlerocket system is atomically updatable. That is, a completely new root filesystem image can
be downloaded into a spare partition. Grub is updated to make the other partition the live one, and
a reboot takes place.

Here is a simplified picture of the partition layout.

```text
┌─────────────────────────────────────────────────────────────────────┐
│                     Bottlerocket Disk Layout                        │
├───────────────────┬───────────────────┬─────────────────────────────┤
│                   │                   │                             │
│         A         │         B         │         User Data           │
│    (Active OS)    │   (Passive OS)    │   (Containers, Settings)    │
│                   │                   │                             │
└───────────────────┴───────────────────┴─────────────────────────────┘
```

**The Update Framework (TUF)**

The Update Framework is used to secure the authenticity of these updates on a Bottlerocket system.
The image is downloaded, and starting with `root.json`, which has been shipped with Bottlerocket as
the root of trust, TUF ensures that whatever is downloaded from the Bottlerocket update repository
is authentic.

**Migrations**

Because the schema of the datastore could change on upgrade or downgrade, Bottlerocket borrowed the
idea of *migrations* from SQL ORM[^2]. In our case, these were little binaries that would run on the
first boot into a new version and take the datastore up or down to the correct version.

During the update process, these binaries are downloaded and stored on the persistent data
partition ready to be executed, if needed, upon the reboot into a different version.

**The Problem: Security**

Prior to the release of Bottlerocket 1.0, these migration binaries were only being authenticated
when they were being downloaded from the TUF repository. There was a TOCTOU vulnerability in which
these binaries could possibly be replaced by an attacker between the time of the download and when
they were needed during reboot. It was understood that this needed to be closed before Bottlerocket
was to be made Generally Available (v1.0).

**The Solution**

The solution was to cache all the TUF repository metadata necessary to re-validate these binaries
during the boot. Then the migrator service needed to be changed to re-validate the binaries and run
them from memory.

**Step 1: Design**

Step 1 was to outline the process fully to the team and to early-adopters of Bottlerocket. I wrote
an *RFC*-style GitHub issue in advance of any changes to describe what we were planning to do.

See: [RFC: Signed Migrations].

**Step 2: Update the TUF Library**

Our Rust TUF library, named [tough], needed to be altered such that it could be told to cache its
metadata and be run in an *unsafe* mode where it ignores expired metadata. This is because the
network is not available during the boot to enact the TUF specification for fetching updated
metadata files.

- [tough: add method for caching a repo with its metadata]
- [tough: support the loading of an expired repo in unsafe mode]
- While were at it, here are all of my [tough PRs].

**Step 3: Change Bottlerocket's Update Systems**

Change the downloader and migrator systems in Bottlerocket to cache the metadata, to re-check the
binary signatures during reboot, and to run them from memory. Also add extensive testing to the
migration system which was a) difficult to test, and b) previously untested.

- [updog: move migration searching code to update_metadata for reuse]
- [updog and migrator: signed migrations]
- [migrator: end-to-end test]

**Step 4: Remove Compat Code**

Because this was a breaking change, we needed to support both modes of operation through a "pivot"
version of Bottlerocket which would point hosts to a new update repository. Once that was done, the
old migration code could be removed.

- [migrator: remove unsigned migration support]

## Bottlerocket Test System

I designed the [Bottlerocket test system]. I used the K-word technology for this. It included an
Operator for driving resource and test state, and two Custom Resource Definitions, one for
Resources (such as EC2 instances, clusters, etc) and one for tests (such as the Sonobouy conformance
suite). Oh yeah, this was done in Rust, not Go, since we were a Rust-based team.

- Here is my [test system Design Document].
- Here are some [cool traits and objects] that I created.

I was proud of my test system's generality when we needed to add testing for bare metal servers.
This was completely foreign from what we had in mind when I designed the framework. But we were able
to add bare metal testing with little or no change to the framework!

- [Bare metal agents]

[^1]: I really don't want the K-word associated with my skill set 😅. I don't mind if we use it to
run our stuff, I just don't want to be a Kubernasties-focused developer.

[^2]: The datastore is not a SQL database. It is just a schema of JSON files, but the idea of
migrations was inspired by SQL ORM solutions such as Ruby on Rails.

<!-- @formatter:off -->

[Bare metal agents]: https://github.com/bottlerocket-os/bottlerocket-test-system/pull/773
[Bottlerocket test system]: https://github.com/bottlerocket-os/bottlerocket-test-system
[Bottlerocket Test System Design Document]: https://github.com/bottlerocket-os/bottlerocket-test-system/blob/develop/docs/DESIGN.md
[Buildsys Refactoring]: https://github.com/bottlerocket-os/twoliter/pull/134
[CLI Bugfix]: https://github.com/bottlerocket-os/twoliter/pull/148
[Change RPM Output Directory Structure]: https://github.com/bottlerocket-os/twoliter/pull/210
[Fargate]: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html#:~:text=Bottlerocket
[First Kit Build]: https://github.com/bottlerocket-os/twoliter/pull/227
[RFC: Signed Migrations]: https://github.com/bottlerocket-os/bottlerocket/issues/905
[Remove Agile Workaround]: https://github.com/bottlerocket-os/twoliter/pull/286
[The Update Framework]: https://theupdateframework.io/
[Tokio and Async Stuff]: https://github.com/bottlerocket-os/twoliter/pull/98
[Twoliter Design Document]: https://github.com/bottlerocket-os/twoliter/tree/develop/docs/design
[Twoliter GitHub Issues]: https://github.com/bottlerocket-os/twoliter/issues?q=is%3Aissue%20state%3Aclosed%20author%3Awebern%20sort%3Acreated-asc
[Twoliter Pull Requests]: https://github.com/bottlerocket-os/twoliter/pulls?q=is%3Apr+author%3Awebern+sort%3Acreated-asc
[Twoliter]: https://github.com/bottlerocket-os/twoliter
[cool traits and objects]: https://github.com/bottlerocket-os/bottlerocket-test-system/tree/develop/agent/resource-agent/src
[migrator: end-to-end test]: https://github.com/bottlerocket-os/bottlerocket/pull/956
[migrator: remove unsigned migration support]: https://github.com/bottlerocket-os/bottlerocket/pull/966
[test system Design Document]: https://github.com/bottlerocket-os/bottlerocket-test-system
[tough PRs]: https://github.com/awslabs/tough/pulls?q=+is%3Apr+sort%3Acreated-asc+author%3Awebern+
[tough: add method for caching a repo with its metadata]: https://github.com/awslabs/tough/pull/111
[tough: support the loading of an expired repo in unsafe mode]: https://github.com/awslabs/tough/pull/121
[tough]: https://github.com/awslabs/tough
[updog and migrator: signed migrations]: https://github.com/bottlerocket-os/bottlerocket/pull/930
[updog: move migration searching code to update_metadata for reuse]: https://github.com/bottlerocket-os/bottlerocket/pull/920
