---
sidebar_position: 2
title: Specification
description: Request for Comment on statechannels framework
keywords: [erc7824, ERC, Ethereum, statechannels, nitro, sdk, development, state channels, ethereum scaling, L2]
tags:
  - erc7824
  - nitro
  - docs
---

# ERC-7824

## Abstract

State Channels is allowing participants to perform off-chain transactions while maintaining the security guarantees of the Ethereum blockchain. The goal is to enhance scalability and reduce transaction costs for decentralized applications.

This standard defines a framework for implementing state channel systems, enabling efficient off-chain transaction execution and dispute resolution. It provides interfaces for managing channel states and an example implementation.

## Motivation

The Ethereum network faces challenges in scalability and transaction costs, making it less feasible for high-frequency interactions. State Channels provide a mechanism to perform most interactions off-chain, reserving on-chain operations for dispute resolution or final settlement. This ERC facilitate the adoption of state channels by standardizing essential interfaces.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

### Glossary of Terms

- **Channel**: A construct where participants interact off-chain using signed messages.
- **Channel ID**: A unique identifier for a channel, derived from the channel's fixed part.
- **Participant**: An entity (EOA) involved in a state channel.
- **Turn Number**: A counter indicating the sequence of moves in the channel.
- **State**: A representation of the channel's condition, updated via signed messages.
- **Fixed Part**: Immutable channel parameters.
- **Variable Part**: Mutable channel parameters that evolve as states are updated.
- **ForceMove**: A procedure for resolving disputes by transitioning the state on-chain.
- **Outcome**: The final result of a channel state used for settling balances.
- **Nitro**: Historial name of the initial protocol

### Data Structures

#### ExitFormat

Standard data structure for exiting an EVM based state channel.

```solidity
/// @title ExitFormat
/// @notice Standard for encoding channel outcomes
library ExitFormat {
    struct SingleAssetExit {
        address asset;
        AssetMetadata assetMetadata;
        Allocation[] allocations;
    }
    
    // AssetMetadata allows for different token standards
    struct AssetMetadata {
        AssetType assetType;
        bytes metadata;
    }
    
    enum AssetType {Default, ERC721, ERC1155, Qualified}
    
    enum AllocationType {simple, withdrawHelper, guarantee}
    
    struct Allocation {
      bytes32 destination;
        uint256 amount;
        uint8 allocationType;
        bytes metadata;
    }
}
```

#### NitroTypes

The `FixedPart`, `VariablePart` compose a "state". The state of the channel is updated, committed to and exchanged between a fixed set of participants.

```solidity
/// @title NitroTypes
/// @notice Defines the core data structures used in state channels.
interface INitroTypes {
    struct FixedPart {
        address[] participants;
        uint64 channelNonce;   // This is a unique number used to differentiate channels
        address appDefinition;  // This is an Ethereum address where a ForceMoveApp has been deployed
        uint48 challengeDuration; // This is duration in seconds of the challenge-response window
    }

    struct VariablePart {
        Outcome.SingleAssetExit[] outcome;
        bytes appData;
        uint48 turnNum;
        bool isFinal;  // This is a boolean flag which allows the channel to be finalized "instantly"
    }
    
    struct SignedVariablePart {
        VariablePart variablePart;
        Signature[] sigs;
    }

    struct RecoveredVariablePart {
        VariablePart variablePart;
        uint256 signedBy; // bitmask
    }
}
```

#### Channel ID

A `channelId` is derived using:

```solidity
bytes32 channelId = keccak256(
    abi.encode(
        fixedPart.participants,
        fixedPart.channelNonce,
        fixedPart.appDefinition,
        fixedPart.challengeDuration
    )
);
```

## State commitments

To commit to a state, a hash is formed as follows:

```solidity
bytes32 stateHash = keccak256(abi.encode(
  channelId,
   variablePart.appData,
   variablePart.outcome,
   variablePart.turnNum,
   variablePart.isFinal
));
```

### Interfaces

#### IForceMoveApp

Define the state machine of a ForceMove state channel protocol

```solidity
/// @title IForceMoveApp
/// @notice Interface for implementing protocol rules in state channels
interface IForceMoveApp is INitroTypes {
     // @notice Encodes application-specific rules for a particular ForceMove-compliant state channel. Must revert or return false when invalid support proof and a candidate are supplied.
     // @dev Depending on the application, it might be desirable to narrow the state mutability of an implementation to 'pure' to make security analysis easier.
     // @param fixedPart Fixed part of the state channel.
     // @param proof Array of recovered variable parts which constitutes a support proof for the candidate. May be omitted when `candidate` constitutes a support proof itself.
     // @param candidate Recovered variable part the proof was supplied for. Also may constitute a support proof itself.
    function stateIsSupported(
        FixedPart calldata fixedPart,
        RecoveredVariablePart[] calldata proof,
        RecoveredVariablePart calldata candidate
    ) external view returns (bool, string memory);
}
```

#### IForceMove

The IForceMove interface defines the interface that an implementation of ForceMove should implement.

```solidity
interface IForceMove is INitroTypes {
    /**
     * @notice Registers a challenge against a state channel. A challenge will either prompt another participant into clearing the challenge (via one of the other methods), or cause the channel to finalize at a specific time.
     * @dev Registers a challenge against a state channel. A challenge will either prompt another participant into clearing the challenge (via one of the other methods), or cause the channel to finalize at a specific time.
     * @param fixedPart Data describing properties of the state channel that do not change with state updates.
     * @param proof Additional proof material (in the form of an array of signed states) which completes the support proof.
     * @param candidate A candidate state (along with signatures) which is being claimed to be supported.
     * @param challengerSig The signature of a participant on the keccak256 of the abi.encode of (supportedStateHash, 'forceMove').
     */
    function challenge(
        FixedPart memory fixedPart,
        SignedVariablePart[] memory proof,
        SignedVariablePart memory candidate,
        Signature memory challengerSig
    ) external;

    /**
     * @notice Overwrites the `turnNumRecord` stored against a channel by providing a candidate with higher turn number.
     * @dev Overwrites the `turnNumRecord` stored against a channel by providing a candidate with higher turn number.
     * @param fixedPart Data describing properties of the state channel that do not change with state updates.
     * @param proof Additional proof material (in the form of an array of signed states) which completes the support proof.
     * @param candidate A candidate state (along with signatures) which is being claimed to be supported.
     */
    function checkpoint(
        FixedPart memory fixedPart,
        SignedVariablePart[] memory proof,
        SignedVariablePart memory candidate
    ) external;

    /**
     * @notice Finalizes a channel according to the given candidate. External wrapper for _conclude.
     * @dev Finalizes a channel according to the given candidate. External wrapper for _conclude.
     * @param fixedPart Data describing properties of the state channel that do not change with state updates.
     * @param candidate A candidate state (along with signatures) to change to.
     */
    function conclude(FixedPart memory fixedPart, SignedVariablePart memory candidate) external;

    // events

    /**
     * @dev Indicates that a challenge has been registered against `channelId`.
     * @param channelId Unique identifier for a state channel.
     * @param finalizesAt The unix timestamp when `channelId` will finalize.
     * @param proof Additional proof material (in the form of an array of signed states) which completes the support proof.
     * @param candidate A candidate state (along with signatures) which is being claimed to be supported.
     */
    event ChallengeRegistered(
        bytes32 indexed channelId,
        uint48 finalizesAt,
        SignedVariablePart[] proof,
        SignedVariablePart candidate
    );

    /**
     * @dev Indicates that a challenge, previously registered against `channelId`, has been cleared.
     * @param channelId Unique identifier for a state channel.
     * @param newTurnNumRecord A turnNum that (the adjudicator knows) is supported by a signature from each participant.
     */
    event ChallengeCleared(bytes32 indexed channelId, uint48 newTurnNumRecord);

    /**
     * @dev Indicates that an on-chain channel data was successfully updated and now has `newTurnNumRecord` as the latest turn number.
     * @param channelId Unique identifier for a state channel.
     * @param newTurnNumRecord A latest turnNum that (the adjudicator knows) is supported by adhering to channel application rules.
     */
    event Checkpointed(bytes32 indexed channelId, uint48 newTurnNumRecord);

    /**
     * @dev Indicates that a challenge has been registered against `channelId`.
     * @param channelId Unique identifier for a state channel.
     * @param finalizesAt The unix timestamp when `channelId` finalized.
     */
    event Concluded(bytes32 indexed channelId, uint48 finalizesAt);
}
```

### Workflows

1. **Opening a Channel**: Participants agree on initial states and open a channel on-chain.
2. **Funding a Channel**: Participants transfer the agreed amount of funds.
3. **Off-Chain Interaction**: Participants exchange signed state updates off-chain.
4. **Dispute Resolution**: If a disagreement arises, a participant can force a state on-chain.
5. **Finalization**: Upon agreement or after a timeout, the channel is finalized, and the outcome is settled.

#### Example Application

The CountingApp contract complies with the ForceMoveApp interface and strict turn taking logic and allows only for a simple counter to be incremented.

```solidity
contract CountingApp is IForceMoveApp {
    struct CountingAppData {
        uint256 counter;
    }

    /**
     * @notice Decodes the appData.
     * @dev Decodes the appData.
     * @param appDataBytes The abi.encode of a CountingAppData struct describing the application-specific data.
     * @return A CountingAppData struct containing the application-specific data.
     */
    function appData(bytes memory appDataBytes) internal pure returns (CountingAppData memory) {
        return abi.decode(appDataBytes, (CountingAppData));
    }

    /**
     * @notice Encodes application-specific rules for a particular ForceMove-compliant state channel.
     * @dev Encodes application-specific rules for a particular ForceMove-compliant state channel.
     * @param fixedPart Fixed part of the state channel.
     * @param proof Array of recovered variable parts which constitutes a support proof for the candidate.
     * @param candidate Recovered variable part the proof was supplied for.
     */
    function stateIsSupported(
        FixedPart calldata fixedPart,
        RecoveredVariablePart[] calldata proof,
        RecoveredVariablePart calldata candidate
    ) external pure override returns (bool, string memory) {
        StrictTurnTaking.requireValidTurnTaking(fixedPart, proof, candidate);

        require(proof.length != 0, '|proof| = 0');

        // validate the proof
        for (uint256 i = 1; i < proof.length; i++) {
            _requireIncrementedCounter(proof[i], proof[i - 1]);
            _requireEqualOutcomes(proof[i], proof[i - 1]);
        }

        _requireIncrementedCounter(candidate, proof[proof.length - 1]);
        _requireEqualOutcomes(candidate, proof[proof.length - 1]);

        return (true, '');
    }

    /**
     * @notice Checks that counter encoded in first variable part equals an incremented counter in second variable part.
     * @dev Checks that counter encoded in first variable part equals an incremented counter in second variable part.
     * @param b RecoveredVariablePart with incremented counter.
     * @param a RecoveredVariablePart with counter before incrementing.
     */
    function _requireIncrementedCounter(
        RecoveredVariablePart memory b,
        RecoveredVariablePart memory a
    ) internal pure {
        require(
            appData(b.variablePart.appData).counter == appData(a.variablePart.appData).counter + 1,
            'Counter must be incremented'
        );
    }

    /**
     * @notice Checks that supplied signed variable parts contain the same outcome.
     * @dev Checks that supplied signed variable parts contain the same outcome.
     * @param a First RecoveredVariablePart.
     * @param b Second RecoveredVariablePart.
     */
    function _requireEqualOutcomes(
        RecoveredVariablePart memory a,
        RecoveredVariablePart memory b
    ) internal pure {
        require(
            Outcome.exitsEqual(a.variablePart.outcome, b.variablePart.outcome),
            'Outcome must not change'
        );
    }
}
```

## Rationale

State channels offer significant scalability improvements by minimizing on-chain transactions. This standard provides clear interfaces to ensure interoperability and efficiency while retaining flexibility for custom protocol rules.

**Scalability**: State channels alleviate these concerns by moving most interactions off-chain, while the blockchain serves as a settlement layer. This construction enable high frequency applications.

**Modular**: Interfaces act as contracts that specify what functions must be implemented without dictating how, allowing developers to create custom implementations suited to specific use cases while maintaining compatibility with the framework.

**Interoperability**: Enable cross-chain interactions, participants can be using two differents chains for opening their channels and perform for example cross-chain atomic swaps.

**Security**: The standard would enable to build an audited framework of primitive which is flexible to accommodate a large number of use cases. ERC-7824 is designed to accommodate this diversity by normalizing common protocol patterns.

## Backwards Compatibility

No backward compatibility issues found. This ERC is designed to coexist with existing standards and can integrate with [ERC-1271](https://eips.ethereum.org/EIPS/eip-1271) and [ERC-4337](https://www.erc4337.io/)

## Security Considerations

This ERC is agnostic of the protocol rules that must be implemented using IForceMoveApp. While the smart-contract framework is a simple set of convention, much of the security risk is moved off-chain. Protocols using State channels must perform security audit of their client and server backend implementation.

## Copyright

Copyright and related rights waived via [CC0](https://github.com/ethereum/ERCs/blob/master/LICENSE.md).
