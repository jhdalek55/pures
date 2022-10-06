# PURE-3

* PURE: 3
* Title: Scudo: Adding Supply Chain Security Capabilities to Uptane
* Version: 1
* Last-Modified: 2022-03-30
* Authors: Marina Moore <>, Aditya Sirish <aditya.sirish@nyu.edu>, Lois Anne Delong <>
* Status: Draft
* Content-Type: <text/markdown>
* Created: 2022-03-30

## Abstract

## Motivation

Software supply chain attacks are on the rise with the
[2021 State of the Software Supply Chain report](https://www.sonatype.com/hubfs/SSSC-Report-2021_0913_PM_2.pdf?hsLang=en-us),
noting a 650% increase in 2021. But, it was the
[2020 Sunburst attack](https://www.cynet.com/attack-techniques-hands-on/sunburst-backdoor-c2-communication-protocol/)
that really made both government and corporate CISOs take action. Affecting more
than 100 companies and nine government agencies, the cause of the attack was
Sunburst malware that had been uploaded via a routine update from Orion software
produced by SolarWinds. The attack taught both public and private concerns some
hard lessons, such as:

* The very thing designed to protect the security of software-patches and
  updates that fix routine vulnerabilities-can also be used to deliver malware
  that can damage or corrupt.
* As software products contain third-party components, program developers are
  not in complete control of the contents of their products.
* Malware can linger undetected for extended periods of time, allowing it to
  infect every system that touches it. The Sunburst malicious code was part of
  an update released by SolarWinds in the spring of 2020, yet it was not
  reported until December of that year.

Automotive software supply chains are particularly tempting targets for
attackers. This is because automotive software runs on a large number of
vehicles and updating all of them to recover quickly from an attack is a
slow and difficult proposition. As such, strategies that can secure automotive
software supply chains are vital.

## Specification

Note: The reader is expected to be familiar with the in-toto specification.

We propose in-toto as an option to achieve software supply chain security
alongside the Uptane deployment. In such a model, in-toto can be backed by
the Uptane root of trust, and use Uptane's delegation structure to distribute
all of in-toto's metadata. Essentially, in-toto metadata is used to attest to
the steps performed in the software supply chain itself, while Uptane is used
to securely distribute that metadata alongside the ECU images.

### Modifications to the Repositories and Vehicle Interactions

In this document, we propose updating one of Uptane's two repositories to store
in-toto metadata alongside the images. The Image repository is recommended as
its contents are typically signed for using offline keys, while the Director
repository usually uses online keys.

When an image is submitted to the Image repository, it must be accompanied by
its in-toto metadata. The in-toto layout and link metadata for every image are
noted in the opaque "custom" field of the image's entry in Targets metadata.
This way, there is no ambiguity about the in-toto metadata to be used when an
image is fetched and verified. Further, as it may not be practical for the top
level Targets role to issue new metadata for every update, Uptane's delegations
model can be used to make this entire process more scalable. Instead, delegated
roles can be used to sign for the image itself and its in-toto link metadata.
However, in-toto layouts and the key artifacts used to verify them should be
signed for by the top level Targets role.

Once the Director repository decides which images are required and signs Targets
metadata for them, the vehicle must fetch the images and all the metadata using
the interfaces provided by both the repositories. In this phase, the Director
repository can continue signing targets metadata for the images, as per the
default functioning of Uptane, and serve the metadata to the automobile. The
automobile fetches the image itself from the Image repository along with its
corresponding in-toto metadata.

TODO: talk about role / delegations setups with a diagram / table

### Full Verification ECUs

The Primary ECU receives images and metadata from the repositories, and sends
them on to the other ECUs that it updates. Each Full Verification ECU is
extended to perform independent in-toto verification that will be secure even
if the Primary ECU is compromised. Some Full Verification ECUs, for example the
Primary, are also expected to perform in-toto verification on behalf of Partial
Verification ECUs that are resource constrained.

For every target image the client needs to install, the corresponding in-toto
layout, links, and keys are identified via the custom field of the existing
Uptane metadata. These files are available as targets on the Image repository.
Therefore, for each installable target, the updater will need to download and
verify the Uptane metadata that signs for the in-toto metadata, either the
Targets role or a delegated role. If verification passes, the actual in-toto
metadata files—layout, links, and public keys—must be downloaded and their
hashes must be compared with the Uptane metadata. With these two checks,
Uptane’s secure delivery properties are extended to the in-toto files. These
files can then be used with confidence by the client to perform in-toto
verification.
### Resource Constrained ECUs

The addition of in-toto metadata generates additional overhead, which can be
problematic for resource constrained ECUs. Uptane already accounts for resource
constrained secondary ECUs by providing a separate Partial Verification
workflow. This document assumes that such ECUs are incapable of performing
in-toto verification and therefore will expect other capable ECUs, perhaps the
Primary ECU, to perform in-toto verification on their behalf. In this model,
while every ECU may not be verifying the software supply chain of the images it
is expected to install, verification is still happening on the individual
automobile. Essentially, there is no difference in the functioning of Partial
Verification secondaries and their Full Verification counterparts.

It is important to note that some automobiles may have no ECUs capable of
providing even a basic level of software supply chain security. Therefore, this
PURE presents a "stop-gap" option. In this configuration, the repository
verifies the software supply chain of the image and signs an in-toto _summary
link_ with the keys used to sign either the top level Targets or delegated
signer role. This way, the key used to sign the summary link metadata is
securely bootstrapped and can be rotated in case of a compromise.

For each target image, the _custom_ field points to the summary link rather than
all the in-toto metadata. The client must download and verify the Targets
metadata for the summary link and, if this passes, download and verify the hash
and signature on the summary link. Once the authenticity of the summary link is
established, the client must verify that the hashes in the products field of the
summary link match those of the image.

The summary link does offer some evidence that the repository performed in-toto
verification on behalf of the automobile, but it is **not** a sufficiently
strong replacement for actual verification on the vehicle. A key issue with the
stop-gap option is that it turns the Image repository into a single point of
failure and therefore can only be trusted as long as the repository remains
secure. As such, this model must only be adopted as a stop-gap option until the
update system can be transitioned to the full model in which in-toto
verification happens on individual vehicles.


## Security Analysis

Scudo is an extension for Uptane that adds in-toto's software supply chain
security properties. Therefore, its threat model is a combination of the threat
models of Uptane for the delivery of the software artifacts, and in-toto for the
development of the image as it passes through the software supply chain.

in-toto is a comprehensive framework that secures each step in the software
supply chain. Furthermore, it plays well with other systems and tools that
attempt to secure specific aspects of the software supply chain. For example,
though in-toto can be used to enforce policies pertaining to the flow of
artifacts through the supply chain, and can provide proof of the people or bots
authorized to make changes at each step, it does not intrinsically provide
mechanisms to store the metadata. Yet, since systems like Grafeas or Sigstore's
Rekor already have support for in-toto, the repository can instead leverage
these systems to store the metadata. Both of these systems already have support
for in-toto. Similarly, in-toto does not provide the information found in a
Software Bill of Materials (SBOM) in a standardized manner. With the importance
of SBOMs growing in the past few years due to both the recognition that knowing
the components in a software artifact is vital to supply chain security and
because they are at the heart of a number of new regulations and standards,
in-toto is working to develop capabilities that can express SPDX and CycloneDX
SBOMs as attestations. This means the distinct security properties of both
in-toto and SBOMs be leveraged. in-toto is also working on capabilities that
can be used to either generate complete SBOMs or to validate that a provided
SBOM is complete.

## Backwards Compatibility

Implementers of Scudo should always consider backwards compatibility. In itself,
this PURE should not introduce any backwards incompatibility to existing Uptane
deployments. However, introducing in-toto into the mix can complicate matters.
Therefore, neither an Uptane nor in-toto implementation should make
backward-incompatible changes to how signatures are generated. This will ensure
that previous package managers are able to continue to install new packages.
Note that TUF can otherwise be used to safely rotate the keys for the entire
system, including those using different key types, key sizes, signature schemes,
and cryptographic hashes. Any backwards incompatible changes that are
unavoidable should be handled with a clear plan, perhaps making use of the
semantics introduced in TAP-14.

## Copyright

This document has been placed in the public domain.
