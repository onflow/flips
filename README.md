# FLIPs (Flow Improvement Proposals)

The purpose of a FLIP is to engage the Flow community in development
by leveraging the collective ideas, insights, and experience of contributors and experts
while also communicating design changes broadly.

Each FLIP describes a protocol change, governance change, or application level standard meant to improve the Flow protocol or ecosystem.
Proposals can be submitted by anyone, and should follow the process below, as well as the [TDB] style guide for each category.

This repository will hold FLIPs to separate them from the [onflow/flow](https://github.com/onflow/flow/tree/master/flips) repository.
This is currently a work in progress, and legacy FLIPs are all in the [flips directory](https://github.com/onflow/flips/tree/main/flips).
These FLIPs will be moved into the appropriate sub-directories over time.

## Application
Application FLIPs are standards for applications built on top of FLOW. This could be token standards, contract interface standards, common design patterns that can benefit from social consensus etc. Application standards should not create protocol changes in themselves, and if they rely on any new protocol features that feature should be written up in its own protocol flip.

## Governance
Governance FLIPs are proposals to take governance actions on the network, for instance changing staking rules, admitting new node operators to the allowlist, adjusting fees, or taking various actions with the the service account.

## Protocol
Protocol FLIPs affect the core Flow protocol. This may include items such as: new algorithms which are required for any flow client to work on the network, payload API changes, cryptographic primitives, etc.

## Cadence
Cadence FLIPs are proposals that affect the Cadence language. If the changes only affect the FVM, the proposal fall under the protocol FLIPs.

# Current FLIPs

- [Application](application):
  - [FLIP 200: Application of BIP 44 in Flow Wallets](application/20201125-bip-44-multi-account.md)
  - [FLIP 636: Non-Fungible Token Metadata](application/20210916-nft-metadata.md)
  - [FLIP 892: Contracts Import Syntax](application/20220323-contract-imports-syntax.md)
  - [FLIP 934: Interaction Templates](application/20220503-interaction-templates.md)
  - [FLIP 1087: Fungible Tokens Metadata](application/20220811-fungible-tokens-metadata.md)
  - [FLIP 45: Flow Client Library (FCL) Specification](application/20221108-fcl-specification.md)
  - [FLIP 69: Fungible Tokens Vault Type Discovery](application/20230206-fungible-token-vault-type-discovery.md)


- [Cadence](cadence):
  - [FLIP 703: Mutability Restrictions](cadence/20211129-cadence-mutability-restrictions.md)
  - [FLIP 718: Cadence Storage API Improvements](cadence/20211208-cadence-storage-api-changes.md)
  - [FLIP 722: Optional References to Indexed Accesses](cadence/20211208-cadence-remove-optional-reference.md)
  - [FLIP 729: Improve Cadence Numeric Supertypes](cadence/20211210-cadence-numeric-supertypes.md)
  - [FLIP 941: Reference Creation Semantics](cadence/20220516-reference-creation-semantics.md)
  - [FLIP 1056: View Functions](cadence/20220715-cadence-purity-analysis.md)
  - [FLIP 11: Attachments](cadence/20220921-attachments.md)
  - [FLIP 13: Change Cadence for-loop semantics](cadence/20221011-for-loop-semantics.md)
  - [FLIP 40: Interface inheritance in Cadence](cadence/20221024-interface-inheritance.md)
  - [FLIP 43: Change the syntax for function types](cadence/20221018-change-fun-type-syntax.md)
  - [FLIP 53: AuthAccount Capabilities](cadence/20221207-authaccount-capabilities.md)

- [Governance](governance):
  - [FLIP GOV-1: Maintain Staking Rewards](governance/20220629-staking-rewards.md)
  - [FLIP GOV-2: Enable Automatic Rewards Payouts](governance/20220718-enable-automatic-rewards.md)
  - [FLIP GOV-3: Dynamic Inclusion Fees](governance/20221005-dynamic-inclusion-fee.md)
  - [FLIP GOV-4: Identifying and Ejecting Errant Access Nodes](governance/20230105-identify-errant-node.md)
  - [FLIP GOV-5: Revisiting Flow storage minimum account balance](governance/20230122-revisiting-storage-fee.md)

- [Protocol](protocol):
  - [FLIP 99: Storage Fees](protocol/20201310-storage-fees.md)
  - [FLIP 324: Self-Ejection (feature for the core protocol)](protocol/20210118-self-ejection.md)
  - [FLIP 343: Handing Messages from Networking Layer to Engines (Core Protocol)](protocol/20210201-consuming-messages-from-network-layer.md)
  - [FLIP 660: Variable Transaction fees](protocol/20211007-transaction-fees.md)
  - [FLIP 677: Execution State Synchronization Protocol](protocol/20211026-state-sync-protocol.md)
  - [FLIP 753: Variable Transaction Fees - Execution Effort I.](protocol/20220111-execution-effort.md)
  - [FLIP 1057: Automated Slot Assignment](protocol/20220719-automated-slot-assignment.md)
  - [FLIP 57: Access Node Staking Minimum](protocol/20230110-accessnode-stake.md)
  - [FLIP 73: Event Streaming API (alpha version)](protocol/20230309-accessnode-event-streaming-api.md)

# Process

## Who is involved?

Everyone is welcome to propose and provide feedback on a FLIP.

A **FLIP author** writes a FLIP and is committed to championing it through the process.

If deemend necessary, a FLIP author may have a **FLIP sponsor** who helps shepherd the
FLIP review process.

A **review committee** is a group of maintainers who are responsible for the strategic direction of components, subcomponents, and public APIs.
They have the responsibility to accept or reject the adoption of the FLIP via a community vote.

## What is a FLIP?

A FLIP is a document that describes a requirement and the proposed changes that
will solve it. Specifically, the FLIP will:

* be formatted according to the FLIP template
* be submitted as a pull request
* be subject to discussion and a review meeting prior to acceptance

## FLIP process

Before submitting a new FLIP, check for prior proposals, the [Flow community forum](https://forum.onflow.org/) and ask in the [Discord server](https://discord.gg/flow).
The idea may have been proposed before or may be in active discussion. Consider contributing or giving feedback to existing proposals.

If a new proposal is appropriate, propose a rough sketch of the idea in the forum [Flow community forum](https://forum.onflow.org/)
and engage in discussion with the community, project contributors, and maintainers, to get early feedback.
Continue to expand the rough sketch into a draft using the FLIP template and further refine the proposal on the forums.

After writing the FLIP draft, gather feedback from project contributors and maintainers before submitting it.

Once the FLIP is ready for review:

1. _(Optional)_ Recruit a sponsor from the maintainers of the project for which your FLIP concerns. A sponsor may help streamline the review process and moderate the discussion with the community.

   Please note that you are not required to recruit a sponsor; this step is entirely optional. If you recruit a sponsor, identify them in the FLIP before posting the PR in step 2.

2. Create an [issue](https://github.com/onflow/flips/issues/new/choose) using the FLIP issue template. The title of the issue should be the title of your FLIP e.g. "Dynamic Inclusion fees".

   Assign an appropriate label for the type of the FLIP (one of `flip:application`, `flip:governance`, `flip:protocol`, `flip:cadence`)

   Submit the issue.

   Note the issue number that gets assigned e.g. for issue https://github.com/onflow/flips/issues/76, the issue number is `76`.
   
3. Create your FLIP as a pull request to this repository ([`onflow/flips`](https://github.com/onflow/flips)).

   Name your FLIP file using the [template](./yyyymmdd-flip-template.md) `YYYYMMDD-descriptive-name.md`,
   where YYYYMMDD is the date of submission, and ‘descriptive-name’ relates to the title of your FLIP.
   For instance, if your FLIP is titled "Event Streaming API",
   you might use the filename `20180531-event-streaming-api.md`.
   If you have images or other auxiliary files,
   create a directory of the form `YYYYMMDD-descriptive-name` in which to store those files.

   Use the issue number generated in step 2 as the FLIP number.

   Mention the FLIP issue by copying the github url or the issue in the comment section.

   At the top of the PR identify how long the comment period will be.
   This should be a minimum of two weeks from posting the PR.

   
5. Send a message to the #developers channel on [Discord](https://discord.gg/flow)
   with a brief description, a link to the PR and a request for review.

6. The author may request a review committee meeting after sufficient discussion has taken place.
   This meeting will include the FLIP author, core contributors and interested community members.
   If the discussion is lively, wait until it has settled before going to review.
   The goal of the review meeting is to resolve minor issues;
   consensus should be reached on major issues beforehand.

7. The meeting may approve the FLIP, reject it, or require changes before it can be considered again.
   FLIPs will be merged into this repository ([`onflow/flips`](https://github.com/onflow/flips))
   with the outcome of the review process (approval, rejection).

8. Implementations of a successful FLIP should reference it in their documentation. While implementation code is not necessary to start the FLIP process, its existence in full or part may help the design discussion.

If in any doubt about this process, feel free to ask on [Discord](https://discord.gg/flow),
the [community forum](https://forum.onflow.org/), or file an issue in this repository
([`onflow/flow`](https://github.com/onflow/flow/issues)).

## Proposal states

* **Proposed:** The FLIP has been proposed and is awaiting review.
* **Rejected:** The FLIP has been reviewed and rejected.
* **Accepted:** The FLIP has been accepted and is either awaiting implementation or is actively being implemented.
* **Implemented (in VERSION):** The FLIP has been implemented.

## Community members

As the purpose of FLIPs is to ensure the community is well represented and served by new changes to Flow,
it is the responsibility of community members to participate in reviewing FLIPs where they are interested in the outcome.

Community members should:

* Provide feedback as soon as possible to allow adequate time for consideration
* Read FLIPs thoroughly before providing feedback
* Be civil and constructive (see [Code of Conduct](CODE_OF_CONDUCT.md))

## Review committees

The constitution of a review committee may change according to the particular governance style and leadership of each project.
For the core Flow Protocol, the committee will exist of contributors to Flow who have expertise in the domain area concerned.

Review committees must:

* ensure that substantive items of public feedback have been accounted for
* add their meeting notes as comments to the PR
* provide reasons for their decisions

If a review committee requires changes before acceptance,
it is the responsibility of the author (and/or sponsor, if applicable) to ensure these are made and seek subsequent approval from the committee members.

## FLIP sponsors (optional)

A sponsor is a Flow maintainer, selected by the author at will, and is responsible for ensuring the best possible outcome of the FLIP process.

In particular, a sponsor may help with the following-

* advocating for the proposed design
* guiding the FLIP to adhere to existing design and style conventions
* guiding the review committee to come to a productive consensus
* if the FLIP is approved and moves to implementation:
  * ensuring proposed implementation adheres to the design
  * liaison with appropriate parties to successfully land implementation

## Keeping the bar high

While we encourage and celebrate every contributor, the bar for FLIP acceptance should be kept intentionally high.
A design may be rejected or need significant revision at any one of these stages:

* initial design conversations on the [Flow community forum](https://forum.onflow.org/) or [Discord server](https://discord.gg/flow)
* critical objections during the feedback phase
* failure to achieve consensus during the design review
* concerns raised during implementation (e.g., inability to achieve backwards
  compatibility, concerns about maintenance appearing once a partial implementation
  is available)

If this process is functioning well, FLIPs are expected to fail in the earlier, rather than later, stages.

## FLIP Template

Use the template [from GitHub](./yyyymmdd-flip-template.md), being sure to follow the naming conventions described above.

## FLIP Evaluation

FLIPs should be evaluated for their impact on the three pillars of Flow. These are:
* **Community** - consider how the FLIP will impact the ability of others to participate in the ongoing design and operation of the Flow network and the applications which depend on it.
* **Empowerment** - consider how the FLIP will improve the economic opportunity for creators, contributors and participants in the community. The FLIP should result in a net positive on the marginal benefits and costs to all the impacted individuals (who choose to register their preference/vote on an issue).
* **Reliability** - and finally, think about how the FLIP will impact the consistency, observability, verifiability, and overall performance of the Flow network for its users.

## Acknowledgements

_Note: the FLIP process and guidelines were adapted from the wonderful RFC process created by the [TensorFlow community](https://github.com/tensorflow/community/tree/master/rfcs)._ ❤️

This work, "Flow Improvement Proposals", is a derivative of "[The TensorFlow RFC process](https://www.tensorflow.org/community/contribute/rfc_process)" by The TensorFlow Authors, used under CC BY 4.0.
