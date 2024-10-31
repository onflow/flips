---
status: draft 
flip: 296
authors: Alex Hentschel (alex.hentschel@flowfoundation.org)
sponsor: Jordan Schalm (jordan.schalm@flowfoundation.org), Yurii Oleksyshyn (yurii.oleksyshyn@flowfoundation.org)
updated: 2024-10-31
---


# [FLIP ???] Utilize Dynamic Protocol State for Version Beacon (coordinating upgrades of the Execution Stack) 

## Objective
![Overview](20241031-execution-stack-versioning/Execution_Stack_Versioning_goal.png)

- Versioning the **Execution Stack** used by ENs, VNs, ANs
    
    Outlook: ANs can decide on whose blocks’ execution states they can run scripts for across Execution HCUs
    

## Terminology

See blog post [[1](https://forum.flow.com/t/protocol-version-upgrade-mechanisms-discussion/5717)] for further details

**Software Version** - The version identifier of a binary distribution of Flow Node software.
By convention, we use semver-ish tag in Git and Docker releases.

Software has bugs. and is frequently incomplete (e.g. API returning ‘not yet implemented’).
The software version is a meaningful reference to describe what the software does in the real world. 

However, we also desire a compact identifier [which we will call the ‘Component Version’] of how a Flow node *should* behave. 

<img src='https://github.com/user-attachments/assets/b88f92ad-c230-417e-bf32-6c9c18e09d61' width='200'>

**Component Version:** version identifier for a component of the flow protocol. It references one specific behaviour of a sub-system (e.g. Execution Stack or HotStuff) of Flow, as prescribed by the protocol. 

In the nutshell, for every block there is one and only one correct way of how to process that block, and how to evolve the execution state. For distributed BFT systems, we need this notion of ‘correct behaviour’, which is inherently implementation agnostic. We want to explicitly express that up to a certain view $v$, we want the protocol to behave in one way and for higher views differently. 

### Relationships between **Software and Component Version**

- Conceptually, for every block, each component of Flow has one and only one component version.
 
- A software version can implement multiple Component Versions.
E.g. AN supporting script execution across HCU boundaries
    
    ❗Don’t couple the software version to the component version!
    

### Reasons we want to move away from existing Version Beacon:

Current Version Beacon:

1. requires that nodes have (potentially long) history (have seen version beacon service event, which is not guaranteed for nodes joining at epoch boundaries)
    
    Better: each block specifies which component version is to be used for processing it
    
2. based on height and hence not usable for upgrading most consensus-related aspects (any many other protocol aspects).
    
    Better: using View instead of height for triggering behaviour changes is generally applicable and more robust (view monotonously increases over time, while height might also decrease).
    

## Dynamic Protocol State already implements better mechanism

💡 In a nutshell, the Protocol State tracks information about each block, including a mechanism to transfer information from the Execution state to the Protocol State in a BFT manner. 

- Flow’s Protocol State to tracks and and provides simple access to information about each blocks (such as epoch number, staking phase, staked nodes allowed to participate as of this block, nodes public keys, etc)  👉 [code](https://github.com/onflow/flow-go/blob/3496c0f02d51602994d4fe60b32fcb00aab084f4/state/protocol/protocol_state.go#L91)
- The Protocol State now also tracks the Component Versions of the most critical consensus component (at the moment: its own version)  👉 [code](https://github.com/onflow/flow-go/blob/3496c0f02d51602994d4fe60b32fcb00aab084f4/state/protocol/protocol_state.go#L100)

☑️ The Protocol State already tracks its own Component Version. You can take a look at these places in the code
 - Protocol State reports its own [version](https://github.com/onflow/flow-go/blob/3496c0f02d51602994d4fe60b32fcb00aab084f4/state/protocol/kvstore.go#L30-L43) as part of every block
 - mechanism for [scheduling version upgrades (at future view)](https://github.com/onflow/flow-go/blob/a6b157ce2770be9356e1cf35d1b0fff63f5e4a76/state/protocol/protocol_state/kvstore/upgrade_statemachine.go#L78-L142) exists
 - mechanism [enforcing that node supports and uses correct](https://github.com/onflow/flow-go/blob/a6b157ce2770be9356e1cf35d1b0fff63f5e4a76/state/protocol/protocol_state/state/protocol_state.go#L235-L248) version as specified by the protocol

<aside>


# Roadmap: Dynamic Protocol State for coordinating Execution Stack upgrades (including Cadence changes)

Biggest change:

- Dynamic Protocol State should ingests Version Beacon Service Event and track’s the Execution Stack’s Component Version

![Illustration of Process](20241031-execution-stack-versioning/Execution_Stack_Versioning_(2).png)

Example:

![Overview](20241031-execution-stack-versioning/Execution_Stack_Versioning_(5).png)



## Semantic versioning

$$
\textnormal{Software Version :}\quad  \underbrace{\texttt{major}\,.\,\texttt{minor}\,}_{\textnormal{Component Version}}.\,\texttt{patch}
$$

- Software with identical  $\texttt{major}.\texttt{minor}$ is cross-compatible irrespective of patch version,
  because they all implement the same specification (represented by the Component Version). Hence, the $\texttt{patch}$ version is not
  part of the Component Version, because it does not influence the conceptual behaviour. Though, it represents implementation details,
  so it is part of the Software Versio.
- In addition, we introduce compatibility requirement from semantic versioning:
    
    $\textnormal{Component Version :} \quad \texttt{major}\,.\,\texttt{minor}$
    
    - Protocol specifications with the same  $\texttt{major}$ must be fully backwards compatible (in practise, mostly additive changes)


## Limitations

For the core protocol, changes are often not backwards compatible. Furthermore, maintaining backwards compatibility can cause
subtle edge cases and drives implementation complexity. 

Lastly, the benefits from backwards compatibility are strong in case we want to process inputs of mixed versions over a prolonged period of time.
In contrast, the protocol abruptly switches at a specific view from one behaviour to another. 


For Flow, controlling complexity and codifying compatibility is essential.
The **compatibility expectations, associated risks and additional complexity** of semantic versioning are not beneficial in *some* areas
of the core protocol. For example the **Component Version** of the Protocol State is specified by a **single integer** (👉[code](https://github.com/onflow/flow-go/blob/c1a1cc0e05f0d323ab2f83dd5d74d8ad486d451e/state/protocol/kvstore.go#L30-L43)). 


**Component Version** guidelines

- Use semantic versioning for areas where
  - benefit from backwards compatibility outweigh the implementation and complexity cost
  - we want to maintain backwards compatability over a longer period of time and across multiple upgrades
- In areas where we can’t easily provide backwards compatibility (e.g. for security or BFT reasons), we should make this explicit by using a single-integer for the Component Version.

## Next Steps

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
    - Breakdown of steps
        
        ![Execution Stack Versioning (3).png](20241031-execution-stack-versioning/Execution_Stack_Versioning_(3).png)
        

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