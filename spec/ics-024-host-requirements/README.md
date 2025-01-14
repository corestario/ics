---
ics: 24
title: Host State Machine Requirements
stage: draft
category: IBC/TAO
requires: 23
required-by: 2, 3, 4, 5, 18
author: Christopher Goes <cwgoes@tendermint.com>
created: 2019-04-16
modified: 2019-08-25
---

## Synopsis

This specification defines the minimal set of interfaces which must be provided and properties which must be fulfilled by a state machine hosting an implementation of the interblockchain communication protocol.

### Motivation

IBC is designed to be a common standard which will be hosted by a variety of blockchains & state machines and must clearly define the requirements of the host.

### Definitions

### Desired Properties

IBC should require as simple an interface from the underlying state machine as possible to maximise the ease of correct implementation.

## Technical Specification

### Module system

The host state machine must support a module system, whereby self-contained, potentially mutually distrusted packages of code can safely execute on the same ledger, control how and when they allow other modules to communicate with them, and be identified and manipulated by a "master module" or execution environment.

The IBC/TAO specifications define the implementations of two modules: the core "IBC handler" module and the "IBC relayer" module. IBC/APP specifications further define other modules for particular packet handling application logic. IBC requires that the "master module" or execution environment can be used to grant other modules on the host state machine access to the IBC handler module and/or the IBC routing module, but otherwise does not impose requirements on the functionality or communication abilities of any other modules which may be co-located on the state machine.

### Paths, identifiers, separators

An `Identifier` is a bytestring used as a key for an object stored in state, such as a connection, channel, or light client. Identifiers MUST consist of alphanumeric characters only. Identifiers MUST be non-empty (of positive integer length).

A `Path` is a bytestring used as the key for an object stored in state. Paths MUST contain only identifiers, constant alphanumeric strings, and the separator `"/"`. 

Identifiers are not intended to be valuable resources — to prevent name squatting, minimum length requirements or pseudorandom generation MAY be implemented, but particular restrictions are not imposed by this specification.

The separator `"/"` is used to separate and concatenate two identifiers or an identifier and a constant bytestring. Identifiers MUST NOT contain the `"/"` character, which prevents ambiguity.

Variable interpolation, denoted by curly braces, is used throughout this specification as shorthand to define path formats, e.g. `client/{clientIdentifier}/consensusState`.

### Key/value Store

The host state machine MUST provide a key/value store interface 
with three functions that behave in the standard way:

```typescript
type get = (path: Path) => Value | void
```

```typescript
type set = (path: Path, value: Value) => void
```

```typescript
type delete = (path: Path) => void
```

`Path` is as defined above. `Value` is an arbitrary bytestring encoding of a particular data structure. Encoding details are left to separate ICSs.

These functions MUST be permissioned to the IBC handler module (the implementation of which is described in separate standards) only, so only the IBC handler module can `set` or `delete` the paths that can be read by `get`. This can possibly be implemented as a sub-store (prefixed key-space) of a larger key/value store used by the entire state machine.

Host state machines MUST provide two instances of this interface -
a `provableStore` for storage read by (i.e. proven to) other chains,
and a `privateStore` for storage local to the host, upon which `get`
, `set`, and `delete` can be called, e.g. `provableStore.set('some/path', 'value')`.

The `provableStore`:

- MUST write to a key/value store whose data can be externally proved with a vector commitment as defined in [ICS 23](../ics-023-vector-commitments). 
- MUST use canonical data structure encodings provided in these specifications as proto3 files

The `privateStore`:

- MAY support external proofs, but is not required to - the IBC handler will never write data to it which needs to be proved.
- MAY use canonical proto3 data structures, but is not required to - it can use
  whatever format is preferred by the application environment.

> Note: any key/value store interface which provides these methods & properties is sufficient for IBC. Host state machines may implement "proxy stores" with path & value mappings which do not directly match the path & value pairs set and retrieved through the store interface — paths could be grouped into buckets & values stored in pages which could be proved in a single commitment, path-spaces could be remapped non-contiguously in some bijective manner, etc — as long as `get`, `set`, and `delete` behave as expected and other machines can verify commitment proofs of path & value pairs (or their absence) in the provable store. If applicable, the store must expose this mapping externally so that clients (including relayers) can determine the store layout & how to construct proofs. Clients of a machine using such a proxy store must also understand the mapping, so it will require either a new client type or a parameterised client.

> Note: this interface does not necessitate any particular storage backend or backend data layout. State machines may elect to use a storage backend configured in accordance with their needs, as long as the store on top fulfils the specified interface and provides commitment proofs.

### Path-space

At present, IBC/TAO utilises the following path prefixes for the `provableStore` and `privateStore`.

Future paths may be used in future versions of the protocol, so the entire key-space in the provable store MUST be reserved for the IBC handler.

Parts of the private store MAY safely be used for other purposes as long as the IBC handler has exclusive access to the specific keys required.
Keys used in the private store MAY safely vary as long as there exists a bipartite mapping between the key formats defined herein and the ones
actually used in the private store implementation.

| Store          | Path format                                                              | Value type        | Defined in |
| -------------- | ------------------------------------------------------------------------ | ----------------- | ---------------------- |
| privateStore   | "clients/{identifier}"                                                   | ClientState       | [ICS 2](../ics-002-client-semantics) |
| provableStore  | "clients/{identifier}/consensusState"                                    | ConsensusState    | [ICS 2](../ics-002-client-semantics) |
| provableStore  | "clients/{identifier}/type"                                              | ClientType        | [ICS 2](../ics-002-client-semantics) |
| provableStore  | "connections/{identifier}"                                               | ConnectionEnd     | [ICS 3](../ics-003-connection-semantics) |
| privateStore   | "ports/{identifier}"                                                     | CapabilityKey     | [ICS 5](../ics-005-port-allocation) |
| provableStore  | "ports/{identifier}/channels/{identifier}"                               | ChannelEnd        | [ICS 4](../ics-004-channel-and-packet-semantics) |
| provableStore  | "ports/{identifier}/channels/{identifier}/key"                           | CapabilityKey     | [ICS 4](../ics-004-channel-and-packet-semantics) |
| provableStore  | "ports/{identifier}/channels/{identifier}/nextSequenceRecv"              | uint64            | [ICS 4](../ics-004-channel-and-packet-semantics) |
| provableStore  | "ports/{identifier}/channels/{identifier}/packets/{sequence}"            | bytes             | [ICS 4](../ics-004-channel-and-packet-semantics) |
| provableStore  | "ports/{identifier}channels/{identifier}/acknowledgements/{sequence}"    | bytes             | [ICS 4](../ics-004-channel-and-packet-semantics) |
| privateStore   | "callbacks/{identifier}"                                                 | ModuleCallbacks   | [ICS 26](../ics-026-routing-module) |

### Module layout

Represented spatially, the layout of modules & their included specifications on a host state machine looks like so (Aardvark, Betazoid, and Cephalopod are arbitrary modules):

```
+----------------------------------------------------------------------------------+
|                                                                                  |
| Host State Machine                                                               |
|                                                                                  |
| +-------------------+       +--------------------+      +----------------------+ |
| | Module Aardvark   | <-->  | IBC Routing Module |      | IBC Handler Module   | |
| +-------------------+       |                    |      |                      | |
|                             | Implements ICS 26. |      | Implements ICS 2, 3, | |
|                             |                    |      | 4, 5 internally.     | |
| +-------------------+       |                    |      |                      | |
| | Module Betazoid   | <-->  |                    | -->  | Exposes interface    | |
| +-------------------+       |                    |      | defined in ICS 25.   | |
|                             |                    |      |                      | |
| +-------------------+       |                    |      |                      | |
| | Module Cephalopod | <-->  |                    |      |                      | |
| +-------------------+       +--------------------+      +----------------------+ |
|                                                                                  |
+----------------------------------------------------------------------------------+
```

### Consensus state introspection

Host state machines MUST provide the ability to introspect their current height, with `getCurrentHeight`:

```
type getCurrentHeight = () => uint64
```

Host state machines MUST define a unique `ConsensusState` type fulfilling the requirements of [ICS 2](../ics-002-client-semantics), with a canonical binary serialisation.

Host state machines MUST provide the ability to introspect their own consensus state, with `getConsensusState`:

```typescript
type getConsensusState = (height: uint64) => ConsensusState
```

`getConsensusState` MUST return the consensus state for at least some number `n` of contiguous recent heights, where `n` is constant for the host state machine. Heights older than `n` MAY be safely pruned (causing future calls to fail for those heights).

Host state machines MUST provide the ability to introspect this stored recent consensus state count `n`, with `getStoredRecentConsensusStateCount`:

```typescript
type getStoredRecentConsensusStateCount = () => uint64
```

### Commitment Path Introspection

Host chains MUST provide the ability to inspect their commitment path, with `getCommitmentPrefix`:

```typescript
type getCommitmentPrefix = () => CommitmentPrefix
```

The result `CommitmentPrefix` is the prefix used by the host state machine's key-value store.
With the `CommitmentRoot root` and `CommitmentState state` of the host state machine, the following property MUST be preserved:

```typescript
if provableStore.get(path) === value {
  prefixedPath = applyPrefix(getCommitmentPrefix(), path)
  if value !== nil {
    proof = createMembershipProof(state, prefixedPath, value)
    assert(verifyMembership(root, proof, prefixedPath, value))
  } else {
    proof = createNonMembershipProof(state, prefixedPath)
    assert(verifyNonMembership(root, proof, prefixedPath))
  }
}
```

For a host state machine, the return value of `getCommitmentPrefix` MUST be constant.

### Port system

Host state machines MUST implement a port system, where the IBC handler can allow different modules in the host state machine to bind to uniquely named ports. Ports are identified by an `Identifier`.

Host state machines MUST implement permission interaction with the IBC handler such that:

- Once a module has bound to a port, no other modules can use that port until the module releases it
- A single module can bind to multiple ports
- Ports are allocated first-come first-serve and "reserved" ports for known modules can be bound when the state machine is first started

This permissioning can be implemented with unique references (object capabilities) for each port (a la the Cosmos SDK), with source authentication (a la Ethereum), or with some other method of access control, in any case enforced by the host state machine. See [ICS 5](../ics-005-port-allocation) for details.

Modules that wish to make use of particular IBC features MAY implement certain handler functions, e.g. to add additional logic to a channel handshake with an associated module on another state machine.

### Datagram submission

Host state machines which implement the routing module MAY define a `submitDatagram` function to submit [datagrams](../../ibc/1_IBC_TERMINOLOGY.md), which will be included in transactions, directly to the routing module (defined in [ICS 26](../ics-026-routing-module)):

```typescript
type submitDatagram = (datagram: Datagram) => void
```

`submitDatagram` allows relayer processes to submit IBC datagrams directly to the routing module on the host state machine. Host state machines MAY require that the relayer process submitting the datagram has an account to pay transaction fees, signs over the datagram in a larger transaction structure, etc — `submitDatagram` MUST define & construct any such packaging required.

### Exception system

Host state machines MUST support an exception system, whereby a transaction can abort execution and revert any previously made state changes (including state changes in other modules happening within the same transaction), excluding gas consumed & fee payments as appropriate, and a system invariant violation can halt the state machine.

This exception system MUST be exposed through two functions: `abortTransactionUnless` and `abortSystemUnless`, where the former reverts the transaction and the latter halts the state machine.

```typescript
type abortTransactionUnless = (bool) => void
```

If the boolean passed to `abortTransactionUnless` is `true`, the host state machine need not do anything. If the boolean passed to `abortTransactionUnless` is `false`, the host state machine MUST abort the transaction and revert any previously made state changes, excluding gas consumed & fee payments as appropriate.

```typescript
type abortSystemUnless = (bool) => void
```

If the boolean passed to `abortSystemUnless` is `true`, the host state machine need not do anything. If the boolean passed to `abortSystemUnless` is `false`, the host state machine MUST halt.

### Data availability

For deliver-or-timeout safety, host state machines MUST have eventual data availability, such that any key/value pairs in state can be eventually retrieved by relayers. For exactly-once safety, data availability is not required.

For liveness of packet relay, host state machines MUST have bounded transactional liveness (and thus necessarily consensus liveness), such that incoming transactions are confirmed within a block height bound (in particular, less than the timeouts assign to the packets).

IBC packet data, and other data which is not directly stored in the state vector but is relied upon by relayers, MUST be available to & efficiently computable by relayer processes.

Light clients of particular consensus algorithms may have different and/or more strict data availability requirements.

## Backwards Compatibility

Not applicable.

## Forwards Compatibility

Key/value store functionality and consensus state type are unlikely to change during operation of a single host state machine.

`submitDatagram` can change over time as relayers should be able to update their processes.

## Example Implementation

Coming soon.

## Other Implementations

Coming soon.

## History

29 April 2019 - Initial draft
11 May 2019 - Rename "RootOfTrust" to "ConsensusState"
25 June 2019 - Use "ports" instead of module names
18 August 2019 - Revisions to module system, definitions

## Copyright

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).
