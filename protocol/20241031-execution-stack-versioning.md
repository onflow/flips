---
status: draft 
flip: 298
authors: Alex Hentschel (alex.hentschel@flowfoundation.org)
sponsor: Jordan Schalm (jordan.schalm@flowfoundation.org), Leo Zhang (leo.zhang@flowfoundation.org), Janez Podhostnik (janez.podhostnik@flowfoundation.org)
updated: 2024-11-01
---


# FLIP 296: Utilize Dynamic Protocol State for Version Beacon (coordinating upgrades of the Execution Stack) 

## Objectives
For a few years, we have been using a mechanism for upgrading Flow's execution stack. In April 2024, we have finished the implementation
of the Dynamic Protocol State, which provides a significantly more robust and generally applicable framework for coordinating upgrades
(for details, please see next section).

This FLIP focuses on the following short term objective, but there is also the goal to foster our long-term development roadmap:
* **Direct goal [focus of this flip]: transition the existing mechanism for upgrading Flow's execution stack to using Dynamic Protocol State**.
  A more detailed scoping is necessary, but my gut feeling is that there is only a limited amount of work needed. In essence, we want to avoid
  extending the old upgrade mechanism in a way that is incompatible with the Dynamic Protocol State and needing to be rewritten later.
* Midterm goal: upgrades of the execution stack are by far the most frequent. By utilizing the Dynamic Protocol State for those upgrades, we
  want to generate learnings on where the new framework should be improved.
* Midterm goal: we are starting to use the Dynamic Protocol State for coordinating upgrades of other parts of the protocol.
  We want to utilize the same framework as for execution stack upgrades, to reduce code and intellectual complexity that would
  otherwise result from using significantly different approaches.    
* Longer-term goal:  Access Nodes [ANs] can decide on whose blocks’ execution states they can run scripts for across different version of the execution stack. 


## Motivation
A blockchain is a distributed system without a central authority controlling the underlying IT infrastructure.
Hence, scenarios must be considered where nodes run different software, node operators potentially don't update in time,
and it is very difficult and time-intensive to coordinate upgrades via means outside the protocol. Hence, we desire
that the protocol itself provides mechanisms to coordinate and enforce behaviors upgrades of honest nodes, whose operators
are responsive on reasonable time scales.

Here, we largely focus on the process by which the protocol specifies how certain component should behave, i.e. the component
specification. Over time, the Flow protocol evolves and the specification for some components changes.
Hence, we version component specifications for ease of referencing them.

![Overview](20241031-execution-stack-versioning/Execution_Stack_Versioning_goal.png)


### Status Quo (as of Nove 2024)
At the moment, we have a mechanism for upgrading Flow's execution stack only. 
This is by far the most frequently updated part of the node software and updates are frequently time-sensitive security fixes.
We have a component called the Version Beacon, which specifies how the implementation _should_ behave and when that "expected behaviour"
is supposed to change. In a nutshell, the Version Beacon for the execution stack currently specifies a block height and a "version" for the
behaviour that should become active when nodes (that are concerned with execution, i.e. ENs, VNs, ANs) reach this height. 
The updates themselves are triggered based on block height, and are therefore called Height-Coordinated Upgrades [HCUs]. 


### Reasons we want to move away from the existing Version Beacon:


Current Version Beacon:
1. focused solely on the software stack for execution, where many aspects would need to be re-implemented if we wanted to apply the same upgrade progress to other parts of the protocol 

   Better: some framework that unifies and encapsulates common functionality for broad protocol upgrades (including but not limited to the execution stack)  

2. requires that nodes have (potentially long) history (have seen version beacon service event, which is not guaranteed for nodes joining at epoch boundaries)

   Better: each block specifies which component version is to be used for processing it

3. based on height and hence not usable for upgrading most protocol-related aspects (such as consensus, processing of information beyond transaction execution).

   Better: using View instead of height for triggering behaviour changes is generally applicable and more robust (view monotonously increases over time, while height might also decrease).


### Dynamic Protocol State provides a framework for more generally applicable and more robust upgrades

💡 In a nutshell, the Protocol State tracks information about each block, including a mechanism to transfer information from the Execution state to the Protocol State in a BFT manner.

- Flow’s Protocol State to tracks and provides simple access to information about each blocks (such as epoch number, staking phase, staked nodes allowed to participate as of this block, nodes public keys, etc) 👉[code](https://github.com/onflow/flow-go/blob/3496c0f02d51602994d4fe60b32fcb00aab084f4/state/protocol/protocol_state.go#L91).
- The Protocol State now also tracks the Component Versions of the most critical consensus component (at the moment: its own version)  👉[code](https://github.com/onflow/flow-go/blob/3496c0f02d51602994d4fe60b32fcb00aab084f4/state/protocol/protocol_state.go#L100).

☑️ The Protocol State already tracks its own Component Version. You can take a look at these places in the code:
- Protocol State reports its own [version](https://github.com/onflow/flow-go/blob/3496c0f02d51602994d4fe60b32fcb00aab084f4/state/protocol/kvstore.go#L30-L43) as part of every block
- mechanism for [scheduling version upgrades (at future view)](https://github.com/onflow/flow-go/blob/a6b157ce2770be9356e1cf35d1b0fff63f5e4a76/state/protocol/protocol_state/kvstore/upgrade_statemachine.go#L78-L142) exists
- mechanism [enforcing that node supports and uses correct](https://github.com/onflow/flow-go/blob/a6b157ce2770be9356e1cf35d1b0fff63f5e4a76/state/protocol/protocol_state/state/protocol_state.go#L235-L248) version as specified by the protocol


:exclamation: For the time being, the Dynamic Protocol State must be updated each time we want to introduce a version for a previously unversioned protocol component.
This is reasonably straight forward and contained engineering work, but not entirely free either. Furthermore, all versioning information
also needs to pass through a smart contract and a specialized service event (this is already the case for the Execution Node's old Version Beacon), which at the moment
are largely strictly typed (for high-assurance purposes).
The scope of this flip is to gain more experience with first applications before discussing details of adding new versions in large numbers.


## User Benefit

Upgrades without significant downtime are very important:
* For large-scale adoption, a good user and developer experience, the flow platform must be reliably available.
* However, we have the need to ship security fixes and evolve Flow, which requires software upgrades. The more frequently
  we can deploy upgrades, the less risk there is for unforeseen problems caused by upgrades. Furthermore, frequent code deployments
  generally reduce engineering efforts. 

# Proposal

## Terminology

See blog post [[1](https://forum.flow.com/t/protocol-version-upgrade-mechanisms-discussion/5717)] for further details:

**Software Version** - The version identifier of a binary distribution of Flow Node software.
By convention, we use semver-ish tag in Git and Docker releases.

Software has bugs and is frequently incomplete (e.g. API returning ‘not yet implemented’).
The software version is a meaningful reference to describe what the software does in the real world. 

However, we also desire a compact identifier [which we will call the ‘Component Version’] of how a Flow node *should* behave. 

<img src='https://github.com/user-attachments/assets/b88f92ad-c230-417e-bf32-6c9c18e09d61' width='200'>

**Component Version:** version identifier for a component of the flow protocol.
It references one specific behaviour of a sub-system (e.g. Execution Stack or HotStuff) of Flow, as prescribed by the protocol.
It is important that the component versioning specifies how a node _should behave_ on the protocol layer.
We say that two specifications are equivalent, if they are describing
the same behaviour - in other words, they are completely interoperable. Equivalent/interoperable component specifications have the same version.

In the nutshell, for every block there is one and only one correct way of how to process that block, and how to evolve the execution state.
For distributed BFT systems, we need this notion of ‘correct behaviour’, which is inherently implementation agnostic.
We want to explicitly express that up to a certain view $v$, we want the protocol to behave in one way and for higher views differently. 

### Relationships between Software and Component Version

- Conceptually, for every block, each component of Flow has one and only one component version.
- A single software version can implement multiple Component Versions.
  E.g. an Access Node supporting script execution across HCU boundaries. Another example is the addition of trigonometric functions to Cadence,
  which we discuss in detail in the section [The subtle notion of downwards compatability in the context of blockchain](#the-subtle-notion-of-downwards-compatability-in-the-context-of-blockchain)

- Similarly multiple software version can implement the _same_ Component Versions.
  For example, consider performance optimizations: if you optimize the memory usage of a node, the node will generally behave exactly the same (just be able to run on a cheaper instance).
  We want to be able to recommend to node operators to upgrade from software version $A$ to $B$, because when running $B$ they only need to pay for smaller instance. But the operators don't have
  to upgrade, as the nodes with the unoptimized and optimized software are fully interoperable. Note that the Component Version specified _how_ a node should behave on the protocol level.
  Differently performant software implementations behaving identically on the protocol layer have the same component version. 


Important: 
* ❗Don’t couple the software version to the component version! We know there will be scenarios where we want one software to implement multiple
  Component Versions and at that point, any one-to-one coupling of Software and Component Version will necessarily break. Instead, for each software
  version, we conceptually have a _list_ of Component Versions that this software supports (even if that list only contains a single element most of the time).
* ❗It is not a requirement that a software version must support all previous component versions.
   - For all nodes (except Access Nodes), we need at most 2 epochs of a history. Their goal is to _extend_ the chain - not act as an archive.
   - We explicitly deviate from Ethereum's practise, where there is one software that can process all blocks since genesis.  In practise, this ability to process all blocks since genesis
     is hardly ever used (impractical time cost). However, it has huge complexity cost for the implementation (littered with if statements, 
     stuck with design patters that were useful long ago but now are more a hindrance for the latest protocol version) and negatively impacts the evolution speed of the overall protocol.    
   - We feel the best approach for Flow is to limit the requirement for downwards compatability for software (in general).
     Containerization provides means run different software versions in parallel should this be desired (e.g. for Archive Nodes).
     This shifts the complexity away from the protocol implementation to the operational layer, where sophisticated widely-adopted solutions already exist.  

# Roadmap: Dynamic Protocol State for coordinating Execution Stack upgrades (including Cadence changes)

Biggest change (and possibly only significant change):

- Dynamic Protocol State should ingest Version Beacon Service Event and track’s the Execution Stack’s Component Version

![Illustration of Process](20241031-execution-stack-versioning/Execution_Stack_Versioning_(2).png)

## Discussion of Possible Versioning Schemes (not exhaustive)

Ultimately, we want to solidify the concept of Component Versions in the protocol to make it easier to evolve the protocol (i.e. the notion of how components _should_ behave).
There exists a variety of possible versioning conventions that we _could_ employ, each with its own tradeoffs between expressing granularity of version changes,
compatability between versions, and complexity.

At the moment, it is not sufficiently evident that there exists a _single_ versioning convention that is _easy and intuitive_ for _all_ the different
components in Flow. Therefore, we recommend that **maintainers** for a specific component **decide** what component **versioning convention** works well for their
component's particular upgrade pattern. 

In the following, we briefly review a few prominent versioning schemes and summarize how these _could_ be used
for component versioning. Though, before we do so, lets look at the notion of downwards compatability, because many versioning schemes encompass some
notion of cross-compatability, which is much more constrained for blockchains compared to traditional IT systems.

### The subtle notion of downwards compatability in the context of blockchain

For example, assume we add trigonometric functions ($\texttt{sin}, \texttt{cos}, \ldots$) to Cadence but don't make any other changes. So we could say
that the new Cadence specification including trigonometric functions is downwards compatible with the prior version without trigonometric functions.
For conventional microservice infrastructures, this is totally fine because the new specification including trigonometric functions can do everything
the previous component specification could. All the existing interactions with the microservice continue to work and if you get lucky to be connected
to a microservice instance that already supports the new spec, you can do additional things. But there is an implicit assumption for conventional
microservice infrastructures that no longer holds for blockchain: _any_ answer you get from _any_ microservice instance is honest. In this scenario,
answers from different microservice instances may differ as long as they differ as they are all honest. For blockchains, we require that _all honest
nodes_ (equivalent of microservice) return _exactly_ the same answer, as we classify nodes as byzantine if they deviate from the majority of identical honest answers.     

Let's look at the following example:
* We have announced that trigonometric functions will be available in Cadence as of view 1000 and later.
* Assume that somebody sends a transaction that computes $\texttt{sin}(\pi)$, which makes it into the block with view 999.
* Some nodes have already updated to the new Cadence version, which in principle can compute $\texttt{sin}(\pi)$, while other node operators haven't upgraded yet. 
* For a blockchain, we need _all_ honest nodes to produce the same result. As our transaction is in block 999, trigonometric functions should not be available yet
  and all honest nodes should fail the transaction. 
* Specifically, this means that the new Cadence version should have two different modes: one with trigonometric functions disabled and one with them enabled. 

So in summary, we need to _precisely_ specify the feature set and the behaviour of each feature. The notion of "at least these features and optionally more"
is insufficient for blockchains.   

Example:

![Overview](20241031-execution-stack-versioning/Execution_Stack_Versioning_(5).png)


### Integer Versioning Scheme

The most basic convention is to use an integer as the version for a particular component (specification). 
When using this convention, we only have a notion of _breaking changes_. For this discussion, it is important to keep in mind that we are
versioning component specifications, i.e. how a particular component _should behave_. We say that two specifications are equivalent, if they are describing
the same behaviour - in other words, they are completely interoperable. Equivalent/interoperable component specifications have the same integer version.
However, whenever the specified behaviour _changes_ or some new functionality is added (that the prior component specification did not include) we have a new version,
because the two specifications are not entirely interoperable anymore. In this case, the Version number is incremented. 

**Applicability and Limitations**

The Integer Versioning Scheme is simple and very intuitive and suits components, where we mostly _change existing behaviours_. This is predominantly the case
for the core protocol, where for example we say that consensus votes previously worked in one way and now work differently.
For example the **Component Version** of the Protocol State is specified by a **single integer** (👉[code](https://github.com/onflow/flow-go/blob/c1a1cc0e05f0d323ab2f83dd5d74d8ad486d451e/state/protocol/kvstore.go#L30-L43)).

With the Integer Versioning Scheme we cannot express the notion of cross-compatability. Nevertheless, Integer Versioning can still be applied to our example
of extending Cadence with trigonometric functions. For example:
* The specification of Cadence without the trigonometric functions could be version `8`.
* Extended with trigonometric functions, we have the specification for Cadence `9`.
* Up to and including view 999, the protocol specifies that a block is to be executed with Cadence `8`:
  * The old Cadence implementation would report that it can execute blocks with Cadence versions `[8]` (slice containing only one version value). 
    The FVM would tell the old Cadence implementation to execute the given block with version `8` and since this version is supported, it proceeds.
    Our transaction fails, because the old Cadence does not know trigonometric functions.
  * The new Cadence implementation would report that it can execute blocks with Cadence versions `[8, 9]`. The difference is that version `8` refuses to
    execute trigonometric functions while version `9` permits them.   
    The FVM would tell the new Cadence implementation to execute the given block with version `8` and since this version is supported, it proceeds.
    Our transaction fails, because the new Cadence knows that in Cadence version `8` trigonometric functions were not yet included. 
*  For blocks wit views 1000 and higher, the protocol specifies that those blocks are to be executed with Cadence `9`:
  * The old Cadence implementation reports that it can execute blocks with Cadence versions `[8]`. However, the FVM would request an 
    execution with version `9`, which is not supported to Cadence errors. 
  * The new Cadence implementation reports that it can execute blocks with Cadence versions `[8, 9]` and since the FVM requests 
    the block's execution with a version that is supported, the block execution proceeds. 


### Semantic Versioning Scheme

A notion of compatibility is at the heart of Semantic Versioning [SemVer]. Though, as discussed before, the notion of downwards compatability must be
carefully analyzed and correctly applied to avoid problems in the blockchain space. Let's revisit our example from before of adding trigonometric functions
to cadence: 
* Let's assume that the Cadence specification without trigonometric functions is denoted by version `1.8`. The spec extended by trigonometric functions is version `1.9`. 
* We have announced that trigonometric functions will be available in Cadence as of view 1000 and later, but a transaction $T$ computing $\texttt{sin}(\pi)$
  was already included in the block with view 999.
* When executing block with view 999, the FVM would tell the new Cadence implementation to execute the block according to Cadence Component Spec version `1.8`, 
  i.e. _without_ trigonometric functions. 
* As you can see from this example, the new Cadence Version should still be able to execute $T$ with trigonometric functions _disabled_. In other words,
  it is insufficient for an implementation of Cadence `1.9` to do everything that `1.8` can and some more - it must be able to _restrict_ its functionality
  to `1.8`. Specifically this additional requirement to _restrict_ the functionality to a prior version goes beyond the established framework of SemVer,
  but it critically required for Flow. 

The advantage of SemVer is that consolidates the concepts of Software Version and Component Version (to some degree):
$$
\textnormal{Software Version :}\quad  \underbrace{\texttt{major}\,.\,\texttt{minor}\,}_{\textnormal{Component Version}}.\,\texttt{patch}
$$

- Software with identical  $\texttt{major}.\texttt{minor}$ is cross-compatible irrespective of patch version,
  because they all implement the same specification (represented by the Component Version). Hence, the $\texttt{patch}$ version is not
  part of the Component Version, because it does not influence the conceptual behaviour. Though, it represents implementation details,
  so it is part of the Software Version.
- In addition, we introduce compatibility requirement from semantic versioning:
    
    $\textnormal{Component Version :} \quad \texttt{major}\,.\,\texttt{minor}$
    
    - Protocol specifications with the same  $\texttt{major}$ must be fully backwards compatible (in practise, mostly limited to additive functionality)
- Additional requirement beyond traditional SemVer: 

  A component implementation supporting $\texttt{major}.\texttt{minor}$ must be able to **restrict its feature** set to _any_ prior version $\texttt{major}.k$
  with $k \leq \texttt{minor}$. 

**Applicability and Limitations**

The benefits from backwards compatibility are strong in case we want to process inputs of mixed versions over a prolonged period of time.
Though, for blockchain, the protocol typically switches abruptly at a specific view from one behaviour to another and never changes back.
Furthermore, maintaining backwards compatibility throughout many $\texttt{minor}$ often causes subtle edge cases and drives implementation complexity. 

For Flow, controlling complexity and codifying compatibility is essential.
The **compatibility expectations, associated risks and additional complexity** of SemVer may not be beneficial in various areas
of the core protocol.
Maintainers considering SemVer for their component version should provide
a strong answer why the Integer Versioning Scheme is disadvantageous for their particular component.    

I (AlexH) do no (yet) see significant benefits of SemVer over Integer Versioning for block execution. Specifically, I don't think it makes any difference
whether the software says that it supports Cadence Specification `[8, 9]`  (Integer Versioning) or `[1.8, 1.9]` (SemVer). The only hypothetical scenario would be,
if we said that _any_ Cadence implementation $1.x$ would _always_ be able to restrict its feature set to any prior version $1.k$ for _any_ $k \leq x$.
However, I consider the requirement of long-tail downwards compatability a risk in itself, due to software and intellectual complexity. So if we start to
introduce cutoffs, e.g. saying that the new Cadence implementation only supports Component Spec Versions `[1.7, 1.8, 1.9]`, then there is little to no difference to
just using integer versions, e.g. the new Cadence implementation specifying that it Component Versions `[7, 8, 9]`.

The only tangible benefit for SemVer I can see is for the Access Node. Here, a client could say that it wants to execute a script on the block with view 999.
Transactions at this view were not allowed to use trigonometric functions. But the client may explicitly request a newer cadence version for its script that already
supports trigonometric functions.  Nevertheless, this is a niche scenario with limited practical relevance. It is not clear whether this scenario warrants the
additional complexity of SemVer with the associated correctness risks (SemVer assumes downwards compatability by default, while maintaining downwards compatability 
in the implementation is generally additional work, so the default assumption of compatability induces additional risks for the happy path of block execution - no a good tradeoff in my opinion).

### Versioning Scheme based on Feature Vectors

For components with frequent addition of entirely new features, we have discussion the importance of _restricting feature sets_ to match prior versions in order to maintain downwards
compatability. At this point, it is intuitive to think about feature flags. For example, we could have a boolean flag $\texttt{Trig} \in \{\texttt{true}, \texttt{false}\}$ indicating
whether trigonometric functions should be accepted as part of a transaction/script.

The following diagram illustrated that Feature Vectors could be represented via Integer versions as well. Here we specifically utilize that the prevalent application pattern for mainnet
is that features are progressively added and enabled.
![Overview](20241031-execution-stack-versioning/Execution_Stack_Versioning_(6).png)


In my [AlexH] opinion, feature flags are an implementation property. If some software exposes three feature flags, e.g. $\texttt{Trig}, \texttt{Ft}_Y, \texttt{Ft}_Z$, the
default expectation is that they can be set in _any_ of the 8 combinations. However, the blockchain scenario is quite different: breaking changes or net additions to Cadence frequently
require a software upgrade for the nodes, which must be announced and adopted by a large fraction of the respective nodes. This process requires days to weeks to make a
version change in mainnet. The very prevalent scenario for mainnet will be that features are progressively added and never again disabled. Most likely, the feature flags will only be
used in very few combinations. With a table we can explicitly represent what combinations of feature flags the current implementation supports (maybe some feature requires another).
To me, this feels a lot more  
If we hypothetically expressed a Component Version via a feature vector (which I am advocating to avoid for most components), we would implicitly commit to allowing _any_ combination
of features to be enables/disabled, which is a very difficult commitment to maintain on the implementation level. 

I feel bug fixes bring versioning via Feature Vectors to its limits: once we learn that we need a bug fix, we would need to add a new feature flag. By exposing the bug fix
as a feature toggle in the Dynamic Protocol State, we are essentially providing a switch to turn the bugfix on and off repeatedly at runtime. For some features this is probably very useful.
However, does it make sense to (figuratively) install a switch in your control board with the sole purpose to turn it on once and never off again? Keep in mind that for every
feature flag, we still need to touch the Protocol State, which is reasonably straight forward and contained engineering work, but not entirely free either. Furthermore, all that
information needs to go through a smart contract and a specialized service event (this is already the case for the Execution Node's old Version Beacon), which additionally 
all would need to support the new feature flag.

I think it would make the most sense in this scenario to bundle changes and roll them out together. In some cases it will be a single bug fix
but in many cases it will be multiple changes combined, because software upgrades on mainnet still need to go through node operators on mainnet for decentralization reasons.
Feature flags are still a great tool for the implementation to provide downwards compatability (e.g. "switch off" bugfixes for older blocks). Nevertheless, I think the fact that
behaviour changes are frequently bundled and form a strict chronological sequence is important. 


## Guidelines and Best Practices for Choosing a Versioning Scheme

### Component Versioning Scheme
- It is recommended that maintainers primarily use Integer Versioning:
  Integer versioning is a comparatively simplistic tool. Though it is also extremely general as it makes effectively no assumptions about usage patterns.
  Low complexity and its generality are strong reasons for using Integer Versioning.

For Flow, controlling complexity and codifying compatibility is essential. While we have little practical experience, there are strong conceptual arguments that
compatibility expectations, associated risks and additional complexity of more sophisticated versioning schemes may not be beneficial in various areas of core protocol.
Maintainers considering schemes other than Integer Versioning should present _convincing reasoning_ in which areas Integer Versioning falls short for their particular component. 

- SemVer may be considered for components:
  - The benefit from backwards compatibility outweigh the implementation and complexity cost.
  - We are happy to commit to _consistently_ creating software that supports _all_ preceding Component Specification Versions.
    Specifically, consider that you are releasing a component specification with version $2.k$, then SemVer makes sense if you can commit that any software
    that supports $2.k$ will _also_ support $2.0, 2.1, \ldots, 2.k-1, 2.k$.
  - In comparison, we recommend to _not_ use SemVer if you are considering software releases that support only a subset of preceding minor versions,
    for example $[2.3, 3.0]$ (but $2.0, 2.1, 2.2$ are not supported). In this case, the software would need to enumerate the supported component versions
    explicitly, or know that all minor versions between $2.8$ (lower bound) and $2.13$ (upper bound) are supported (which only works within one major version).
    In this case, Integer Versioning is equally useful (for enumerating supported component versions, or providing an interval of supported component versions). 
    The additional structure of SemVer is more of a hindrance than a help.

### Software Versioning

This FLIP makes no suggestions how to version software (on purpose). However, it is critically important to:
* decouple component version from software versions 
* A single component software can implement multiple component versions. Hence, each software must itself know which component versions it implements.

### Enforcing that Software is compatible with a specified component Version  

* A component software must _automatically verify_ that it runs a compatible version:
   * In most cases, nodes run the Consensus Follower component, which emits notifications when a new block was ingested, finalized etc. 
   * Component software generally subscribes and listens to those notifications and then determines its respective tasks.
   * At this point we have a block and tasks resulting from knowing this block. A component typically includes its own way of queueing and executing the resulting tasks.
   * For components versioned through the Dynamic Protocol State, the component software should check for each task that it implements the appropriate component version specified by the block corresponding to the task.    
   * At the moment, we focus on components, whose tasks are associated to blocks. That is not the case for all components (a counter example is the networking layer) - though _the vast majority_ of tasks
    (especially Cadence execution) are associated with blocks.    
   * The first time when a node checks compatability between software and component version is when it is bootstrapped from a Sealing Segment.
     The Sealing Segment contains blocks, blocks specify component versions and the software reading the sealing segment needs to support the specified component versions.
     If that is not the case, node instantiation must fail.
     Hence, nodes that successfully bootstrapped have confirmed that their software is compatible with at least the blocks in the Sealing Segment. As they ingest blocks,
     nodes learn about _upcoming_ version changes, before those version changes take effect, through blocks they are guaranteed to be compatible with.
     The edge case that a node might miss the information about an upcoming version upgrade does not exist (a major advancement of the Dynamic Protocol State over the old version beacon).
   * Note that the proposal only provides a mechanism to retrieve the current component version. If the software version does not support this component version,
     the proposed mechanism does not provide any work around. This is on purpose, because for the protocol to make progress, software needs to implement the respective protocol requirements.
     Being incompatible is a failure scenario. In such a case, it is up to the component to decide how to proceed (e.g. return an exception, crash, etc.)
* With the introduction of new version, it is important to keep track and document them. For human readability, we recommend a light-weight process, like a changelog.
  However, the software must also provide algorithmic means to determine whether it can process a specific task according to specified component version.
  - For a software, human-readable information about what component versions it support is important for node operators:
    The node operator needs to know which software version they have to bootstrap their node with (depending on the component versions specified by the blocks in the Sealing Segment). 
    If they bring up a node starting from a sealing segment far into the past, the node will catch up and might quickly learn that it requires yet another upgrade and then automatically stop/crash.
    It is helpful for the node operator to know this upfront to plan further software upgrades.
  - Note that the human-readable versioning information is helpful for the node operator to do the right thing (use compatible software). But human-readable versioning information in insufficient to
    guarantee that node operators will do the right thing. Hence, the software must additionally automatically check that is supports the specified component version for every block it processes and
    all tasks associated with hat block.    

## Comparison to other approaches in the blockchain industry

💡Note that this approach is significantly different from Ethereum's. In Ethereum, it is common practise to hard-code the block heights / views at which features
change directly into the implementation. For Flow, we have the Dynamic Protocol State as an interim abstraction layer: 
* the protocol state translates view to version
* the component implementation utilizes the version to determine what features to enable / disable

Flow's approach is a _major_ advancement over Ethereum's approach in the following regard:
* Ethereum assumes forward compatability by default. In other words, unless the implementation knows that the protocol specification will change at a certain point in the future,
  it assumes that everything stays the same. So nodes whose software is not upgraded don't recognize themselves that they are incompatible to a future version.
  Hence, it requires manual human action (software upgrade) for the nodes to behave correctly and in the absence of human intervention, nodes will behave incorrectly (continue with the old version).
  Therefore, it is hard for the Ethereum protocol to evolve, because the default is to continue with the current functionality. Ethereum addressed this challenge with the "Difficulty Bomb", an
  arguably crude an-on to the protocol with limited effectiveness. 
* In contrast, Flow's versioning mechanism through the Dynamic Protocol State works the opposite: the protocol communicates _upfront_ to the implementation that the specified honest behaviour will
  change at some specific point in the future. This information about upgrades is distributed upfront through the Dynamic Protocol State and therefore also accessible to old implementations
  that have not yet upgraded. This inverts the default and facilitates protocol evolution: by default, the protocol switches to the new behaviour and nodes not supporting the new behaviour will
  proactively stop participating (as opposed to continuing by default with the old version).  

# Next Steps

Challenge: missing seed for starting engineering work:

- complex topic, spanning three areas of Flow [execution, Protocol, Data Availability]
- everyone is worried they are missing something, but we have limited time / priority to flesh out a holistic vision for versioning each and every aspect of the protocol  
- keep talking about it, bigger picture remains hazy, 
so ICs keep extending our existing but insufficiently general solution (existing Version Beacon, solely based on service events, where tracking and complying to service events is entirely left to the implementation)

***Approach*:**

**We work towards transitioning the _existing_ Version Beacon to use the Protocol State.**

- Decide now what convention we use for:
    - Component Version format of Execution Stack? (e.g. $\texttt{major}.\texttt{minor}$ or single integer? Currently, the Version Beacon uses semver,
      but Bastian thinks this might be unnecessary complex for Cadence)
    - One Component Version for Cadence only?
        
        or one Component Version for Cadence+FVM combined?
        
        or two separate Component Versions (one for Cadence and one for FVM)  
        
- Start by including Component Version for Execution Stack into Dynamic Protocol State
  
  Breakdown of steps:
  ![Execution Stack Versioning (3).png](20241031-execution-stack-versioning/Execution_Stack_Versioning_(3).png)
        

## Outlook

[Bastian recommended](https://github.com/onflow/flips/pull/296#discussion_r1825273426) that we specify a component version for the "Cadence Language",
which specifies the available language features and their behaviour. This is useful when incrementally adding new features, where it is relatively trivial
to have one binary supporting the different feature sets.

In addition, [Bastian recommended](https://github.com/onflow/flips/pull/296#discussion_r1825273426) to consider Component Versions for subcomponents
of Cadence/FVM, such as "storage format" (how are objects stored in account storage encoded?),
"CCF format" and
"JSON format" (how are "external" objects encoded?).




## Further reading

- [[1](https://forum.flow.com/t/protocol-version-upgrade-mechanisms-discussion/5717)] Jordan’s [**Protocol Version Upgrade Mechanisms Discussion**](https://forum.flow.com/t/protocol-version-upgrade-mechanisms-discussion/5717) (flow forum post)
- [Core-Protocol WG Meeting on Versioning, May 23, 2024](https://github.com/onflow/Flow-Working-Groups/blob/main/core_protocol_working_group/meetings/2024-05-23_Versioning_sub-working-group.md)
- [*[Brainstorming] HCU-style upgrades for all node roles* ](https://www.notion.so/Brainstorming-HCU-style-upgrades-for-all-node-roles-b6b0ab084075432782cd0407b73479c7?pvs=21)

# Question and Answers:

### Do we want to track a Component Version for _every_ component of Flow? 

**Answer**: For most components, we do _not_ track their version explicitly. Necessary updates are infrequent and not time sensitive, so that we
can just bundle all changes across many components and ship them all together as part of a major upgrade (aka Spork).

However, for very few components, upgrades are frequent and time sensitive (e.g. security fixes in Cadence), so that we cannot
wait for the major upgrade (aka Spork). In that case, we want to deploy the upgrades into the running network and need to specify
what should happen (i.e. the component version) and when that is going to change. Only for those components we want to track their component version
via the protocol state. 