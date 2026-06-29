---
title: Release Guide
---

# Apache Auron (Incubating) Release Guide

This guide describes how to publish an official Apache Auron (Incubating) release. It is
intended for the **Release Manager** — a committer chosen by the community to drive a
particular release — and for community members who want to verify and vote on a release
candidate.

Because Auron is currently incubating at the Apache Software Foundation, every release
must be approved by **two** votes: first in the Auron community (the PPMC), and then in
the Apache Incubator (the IPMC). A release is only official once both votes have passed.

If anything here conflicts with official Apache policy, the ASF policy always wins.
Please read the following references before your first release:

- [Apache Release Policy](https://www.apache.org/legal/release-policy.html)
- [Incubator Release Management Guide](https://incubator.apache.org/guides/releasemanagement.html)
- [Incubator Release Checklist](https://cwiki.apache.org/confluence/display/INCUBATOR/Incubator+Release+Checklist)
- [How to verify a release](https://www.apache.org/info/verification.html)

## Overview

A full release goes through the following stages:

1. **Prepare** — bump the version, tag a release candidate (RC), and draft release notes.
2. **Build & publish the RC** — produce the signed source tarball and upload it to the
   Apache `dev` (staging) SVN repository.
3. **Vote in the Auron community** — start a `[VOTE]` thread on `dev@auron.apache.org`.
4. **Vote in the Incubator** — forward the approved RC to `general@incubator.apache.org`.
5. **Finalize** — move the artifacts to the Apache `release` SVN repository.
6. **Announce** — send the release announcement and update the website.

If a vote fails, fix the problem, increase the RC number, and start again from the
**Prepare** stage.

## Prerequisites

These steps only need to be done once. Subsequent releases can reuse the same setup.

### 1. Apache account

You need an Apache committer account (your Apache ID). The finalize step additionally
requires PMC (PPMC) privileges, because it writes to the Apache `release` repository.

### 2. Tools

Make sure the following tools are installed on a macOS or Linux machine:

- **Git**
- **Subversion (SVN)** — Apache distributes release artifacts via SVN (`dist.apache.org`).
- **GnuPG (GPG) 2.x** — to sign the release artifacts.
- **JDK** (8 / 11 / 17) and **Maven** — to build Auron from source. See the
  [Getting Started](/documents/getting-started) guide for the full build toolchain.

### 3. Generate and publish your GPG key

Release artifacts must be signed with your personal GPG key.

1. If you do not have a key yet, generate one (RSA, 4096 bits). Use the email address
   associated with your Apache account:

   ```shell
   gpg --full-generate-key
   ```

2. Find your key id and export the public key:

   ```shell
   gpg --list-keys --keyid-format SHORT
   gpg --armor --export <your-key-id>
   ```

3. Append your public key to the Auron `KEYS` files in both the `dev` and `release`
   distribution repositories. Editing the `release` `KEYS` file requires a PMC member.

   - `https://dist.apache.org/repos/dist/dev/incubator/auron/KEYS`
   - `https://dist.apache.org/repos/dist/release/incubator/auron/KEYS`

   For example:

   ```shell
   svn checkout https://dist.apache.org/repos/dist/release/incubator/auron auron-dist
   cd auron-dist
   (gpg --list-sigs <your-key-id> && gpg --armor --export <your-key-id>) >> KEYS
   svn commit -m "Add GPG key for <your apache id>"
   ```

### 4. Environment variables

The release scripts read three environment variables:

| Variable        | Description                                          |
|-----------------|------------------------------------------------------|
| `ASF_USERNAME`  | Your Apache ID (committer username).                 |
| `ASF_PASSWORD`  | Your Apache account password.                        |
| `RELEASE_RC_NO` | The release candidate number, starting from `0`.     |

## Prepare the Release

All commands in this section are run from the root of a clean checkout of the
[`apache/auron`](https://github.com/apache/auron) repository.

### 1. Bump the version

Releasing requires two pull requests that change `project.version` in `pom.xml`:

1. Drop the `-SNAPSHOT` suffix and set the release version. For an incubating project the
   release version **must** contain the word `incubating`:

   ```
   [RELEASE] Bump version 6.0.0-incubating
   ```

2. Bump to the next development (`SNAPSHOT`) version:

   ```
   [RELEASE] Bump version 7.0.0-SNAPSHOT
   ```

> Each release attempt (a new RC) needs its own version-bump commits, so you may end up
> with several such commits over the course of a release.

### 2. Cut a release branch (recommended)

Consider cutting a `branch-X.Y` (for example `branch-6.0.0`) from the release commit so
that later hotfixes can be applied without disturbing the main branch.

### 3. Tag the release candidate

Create a tag on the release commit. The tag follows the pattern `vX.Y.Z-rcN`, where `N`
is the RC number. Increase `N` for every new attempt:

```shell
git tag v6.0.0-rc0
git push origin v6.0.0-rc0
```

### 4. Draft the release notes

Generate the change log by comparing the previous release tag with the new one using
GitHub's compare view, for example:

```
https://github.com/apache/auron/compare/v5.0.0...v6.0.0-rc0
```

Go through every commit and compile the list of contributors — both the authors of the
pull requests and the people who reviewed them should be acknowledged. The release notes
are published later on the website's [Archives](/archives/all-releases) page.

## Build & Publish the Release Candidate

Auron releases a **source-only** distribution. The `publish` step builds the source
tarball, signs it with your GPG key, generates a SHA-512 checksum, and uploads everything
to the Apache `dev` (staging) SVN repository:

```shell
export RELEASE_RC_NO=0
./build/release/release.sh publish
```

This produces and uploads the following artifacts to
`https://dist.apache.org/repos/dist/dev/incubator/auron/<release-tag>/`:

- `apache-auron-<version>-source.tgz` — the source release tarball
- `apache-auron-<version>-source.tgz.asc` — the GPG signature
- `apache-auron-<version>-source.tgz.sha512` — the SHA-512 checksum

After uploading, double-check that the files appear in the staging repository and that the
signature and checksum verify correctly (see the verification checklist below).

## Vote in the Auron Community

Start a vote on the `dev@auron.apache.org` mailing list. The vote must stay open for at
least **72 hours** and needs at least **three binding `+1` votes** (from PPMC members)
and **no `-1`** votes. Anyone may vote or join the discussion, but only PPMC votes are
binding in the Auron community. If there is a `-1`, resolve the concern and, if needed,
restart the vote.

### `[VOTE]` email template

```text
Title: [VOTE] Release Apache Auron (Incubating) <release-tag>

Hi Auron community,

This is a call for vote to release Apache Auron (Incubating) <release-tag>

The git tag to be voted upon:
https://github.com/apache/auron/releases/tag/<release-tag>

The git commit hash:
<commit-hash>

The source artifacts can be found at:
https://dist.apache.org/repos/dist/dev/incubator/auron/<release-tag>

Fingerprint of the PGP key release artifacts are signed with:
<your-key-id>

My public key to verify signatures can be found in:
https://dist.apache.org/repos/dist/dev/incubator/auron/KEYS (Apache ID: <your-apache-id>)

The vote will be open for at least 72 hours or until necessary
number of votes are reached.

Please vote accordingly:

[ ] +1 approve
[ ] +0 no opinion
[ ] -1 disapprove (and the reason)

Checklist for release:
https://cwiki.apache.org/confluence/display/INCUBATOR/Incubator+Release+Checklist
Steps to validate the release:
https://www.apache.org/info/verification.html

Starting with my +1 (binding):

* Download links, checksums and PGP signatures are valid.
* Source code distributions have correct names matching the current release.
* Release files have the word incubating in their name.
* DISCLAIMER, LICENSE and NOTICE files are correct.
* All files have license headers if necessary.
* No unlicensed compiled archives bundled in source archive.
* The source tarball matches the git tag.
* Build from source is successful.

Thanks,
<release manager>
```

### How to verify a release candidate

Voters are encouraged to check the items below before casting their vote. Download the
tarball, signature, checksum, and the `KEYS` file from the staging repository, then:

```shell
# Import the project's public keys
curl -O https://dist.apache.org/repos/dist/dev/incubator/auron/KEYS
gpg --import KEYS

# Verify the GPG signature
gpg --verify apache-auron-<version>-source.tgz.asc apache-auron-<version>-source.tgz

# Verify the checksum
shasum -a 512 -c apache-auron-<version>-source.tgz.sha512
```

A reviewer's checklist (see the [Incubator Release Checklist](https://cwiki.apache.org/confluence/display/INCUBATOR/Incubator+Release+Checklist)
for the full version):

- Release files are in the correct staging location.
- Digital signature and hashes are correct.
- The release file name contains `incubating`.
- `DISCLAIMER`, `LICENSE`, and `NOTICE` files exist and are correct.
- All source files carry the appropriate license headers.
- No unlicensed compiled archives are bundled in the source archive.
- The contents of the release match the corresponding tag in Git.
- The release can be built from source.

A typical reply when voting:

```text
+1 (binding)

I checked:
- release files in the correct location
- digital signature and hashes are correct
- DISCLAIMER is fine
- LICENSE and NOTICE files exist and are correct
- contents of the release match the tag in VCS
- can build the release from the source

Thanks,
<name>
```

### `[RESULT][VOTE]` email template

After 72 hours, close the vote with a result email:

```text
Title: [RESULT][VOTE] Release Apache Auron (Incubating) <release-tag>

Hi Auron community,

The vote closes now as 72hr have passed. The vote PASSES with:

(* = binding)

+1:
<PPMC member 1> *
<PPMC member 2> *
<community member>

There are no 0 or -1 votes.

The voting thread:
https://lists.apache.org/thread/<auron-vote-thread>

I will now bring the vote to general@incubator.apache.org to get
approval by the IPMC.
If this vote passes, the release is accepted and published.

Thanks,
<release manager>
```

## Vote in the Incubator

Once the community vote passes, request approval from the Incubator PMC by starting a new
vote on `general@incubator.apache.org`. As before, the vote stays open for at least **72
hours** and needs at least **three binding `+1` votes** — here from IPMC members.

### `[VOTE]` email template (Incubator)

```text
Title: [VOTE] Release Apache Auron <version>-incubating-rcN

Hello Incubator Community,

This is a call for a vote to release Apache Auron (Incubating) version
<version>-incubating-rcN

The Apache Auron community has voted on and approved a proposal to release
Apache Auron (Incubating) version <version>-incubating-rcN

We now kindly request the Incubator PMC members review and vote on this
incubator release.

Auron community vote thread:
 • https://lists.apache.org/thread/<auron-vote-thread>

Vote result thread:
 • https://lists.apache.org/thread/<auron-vote-result-thread>

The release candidate:
 • https://dist.apache.org/repos/dist/dev/incubator/auron/<release-tag>/

Git tag for the release:
 • https://github.com/apache/auron/releases/tag/<release-tag>

Public keys file:
 • https://dist.apache.org/repos/dist/release/incubator/auron/KEYS

The change log is available in:
 • https://github.com/apache/auron/compare/<previous-tag>...<release-tag>

The vote will be open for at least 72 hours or until the necessary number
of votes are reached.

Please vote accordingly:
 [ ] +1 approve
 [ ] +0 no opinion
 [ ] -1 disapprove with the reason

More detailed checklist please refer:
• https://cwiki.apache.org/confluence/display/INCUBATOR/Incubator+Release+Checklist

Steps to validate the release, please refer to:
• https://www.apache.org/info/verification.html

Thanks,
On behalf of Apache Auron (Incubating) community
```

### `[RESULT][VOTE]` email template (Incubator)

```text
Title: [RESULT][VOTE] Release Apache Auron <version>-incubating-rcN

Hello Incubator Community,

The vote to release Apache Auron (Incubating) <version>-incubating-rcN has passed
with <n> +1 and no +0 or -1 votes.

(* = binding)

+1:
<IPMC member 1> (binding)
<IPMC member 2> (binding)
<community member>

The voting thread:
https://lists.apache.org/thread/<incubator-vote-thread>

Thanks for reviewing and voting for our release candidate.
We will proceed with publishing the approved artifacts and sending
out the announcement soon.

Regards,
<release manager>
```

## Finalize the Release

::: warning IRREVERSIBLE
The `finalize` step moves the artifacts into the public Apache `release` repository and
cannot be undone. Make sure **both** votes have passed and that you are finalizing the
correct RC. This step must be run by a PMC (PPMC) member.
:::

```shell
export RELEASE_RC_NO=0
./build/release/release.sh finalize
```

This moves the artifacts from the staging repository to
`https://dist.apache.org/repos/dist/release/incubator/auron/auron-<version>/`. It can take
up to 24 hours for the release to propagate to the Apache mirror network.

Then create the official GitHub release for the tag at
`https://github.com/apache/auron/releases`, attaching the release notes drafted earlier.

## Announce

Send the announcement to `dev@auron.apache.org` and `announce@apache.org` (cc
`general@incubator.apache.org`). Send the announcement only after the artifacts are
available on the Apache mirrors.

### `[ANNOUNCE]` email template

```text
Title: [ANNOUNCE] Apache Auron (Incubating) <version> available

Hi all,

Apache Auron (Incubating) community is glad to announce the
new release of Apache Auron (Incubating) <version>.

Auron is a high-performance, native accelerator for big data engines.
It supports Apache Spark full-featured and improves the performance,
stability and elasticity of Spark jobs.

Download Link: https://auron.apache.org/archives/v<version>-incubating.html
(Starting from v6.0.0, Auron only provides source download because there are
many new build options. Users can build with their own environment and options,
or build on GitHub Actions by forking the repository.)

GitHub Release Tag:
• https://github.com/apache/auron/releases/tag/<release-tag>

Release Notes:
• https://auron.apache.org/archives/v<version>-incubating.html

Website: https://auron.apache.org/

Auron Resources:
• Issues: https://github.com/apache/auron/issues
• Mailing list: dev@auron.apache.org

<release manager>
On behalf of the Apache Auron (Incubating) community
```

## Post-Release

After the release is announced, finish the following housekeeping:

1. **Update the website** — add the new version to the [Archives](/archives/all-releases)
   page with its release notes and download links.
2. **Clean up old release candidates** — remove obsolete RC directories from the `dev`
   staging repository:

   ```shell
   svn rm https://dist.apache.org/repos/dist/dev/incubator/auron/<old-release-tag> \
     -m "Remove obsolete RC for Apache Auron <version>"
   ```

3. **Keep only the latest releases** in the `release` repository — the Apache mirror
   system only needs the current releases; older ones are served from the
   [archive](https://archive.apache.org/dist/incubator/auron/).

## References

- [Apache Release Policy](https://www.apache.org/legal/release-policy.html)
- [Incubator Release Management Guide](https://incubator.apache.org/guides/releasemanagement.html)
- [Incubator Release Checklist](https://cwiki.apache.org/confluence/display/INCUBATOR/Incubator+Release+Checklist)
- [How to verify a release](https://www.apache.org/info/verification.html)
