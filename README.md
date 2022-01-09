---
eip: <to be assigned>
title: Consensus Layer Withdrawal Protection
description: Provides additional security in the "change withdrawal address" operation when a consensus layer mnemonic may be compromised, without changing consensus
author: Benjamin Chodroff (@benjaminchodroff), Jim McDonald (@mcdee)
discussions-to: <URL>
status: Draft
type: Standards Track
category (*only required for Standards Track): Interface
created: 2022-01-09
requires (*optional): <EIP number(s)>
---

This is the suggested template for new EIPs.

Note that an EIP number will be assigned by an editor. When opening a pull request to submit your EIP, please use an abbreviated title in the filename, `eip-draft_title_abbrev.md`.

The title should be 44 characters or less. It should not repeat the EIP number in title, irrespective of the category. 

## Abstract
The consensus layer change withdrawal credentials proposal is secure for a single user who has certainty their keys and mnemonic have not been compromised. However, as withdrawals on the consensus layer are not yet possible, no user can have absolute certainty that their keys are not compromised until the change withdrawal address is on chain, and by then too late to change. In a situation where multiple users have access to the consensus layer withdrawal private key, it is impossible for the network to differentiate the “legitimate” holder of the key from the “illegitimate”. However, there are signals that could be considered in a wider sense without changing core ethereum consensus. The most verifiable one is that the legitimate holder was also in control of the execution layer deposit address. Clients and node operators may optionally load a list of verifiable deposit addresses, a list of verifiable change withdrawal address messages to help broadcasts, and specify a rebroadcast delay that may create a social consensus for legitimate holders. 

## Motivation
The motivation section should describe the "why" of this EIP. What problem does it solve? Why should someone want to implement this standard? What benefit does it provide to the Ethereum ecosystem? What use cases does this EIP address?

The Consensus Layer change withdrawal credentials operation is [not yet fully specified](https://github.com/ethereum/consensus-specs/pull/2759), but is likely to have at least the following fields:
* Validator index
* Current withdrawal public key
* Proposed execution layer withdrawal address
* Signature by withdrawal private key over the prior fields
	
Setting a withdrawal address to an execution layer address was not supported by the eth2.0-deposit-cli until v1.1.1 on March 23, 2021, leaving some early adopters wishing they could force change their execution layer address earlier. Forcing this change is not something that can be enforced in-protocol, partly due to lack of information on the beacon chain about the execution layer deposit address and partly due to the fact that this was never listed as a requirement. It is also possible that the execution layer deposit address is no longer under the control of the legitimate holder of the withdrawal private key. 

However, it is possible for individual nodes to locally restrict the changes they wish to include in blocks they propose, and which they propagate around the network, in a way that does not change consensus. It is also possible for client nodes to help broadcast signed change withdrawal address requests to ensure as many nodes witness this change as soon as possible in a fair manner. Further, such change withdrawal address signed messages can be preloaded into clients in advance to further help nodes filter attacking requests.

This proposal describes three optional mechanisms which a client software may optionally implement, and end users may optionally adopt:
1. A "Change Withdrawal Address Rebroadcast Delay", which instructs clients to delay rebroadcasting change withdrawal messages when they do not match a provided signature, or the original deposit address
2. A list of accepted change withdrawal addresses signed messages to broadcast once the network supports the operation, and immediately rebroadcast if received, and drop/not share if mismatching
3. A list of withdrawal credentials and deposit addresses verifiable against the execution layer to act as a tiebreaker in the event of receiving conflicting messages (deposit address should be accepted without delay, whereas new addresses should have a rebroadcast delay)

This proposal provides optional additional protection. It aims to request nodes set a priority on withdrawal credential claims that favour a verifiable execution layer deposit address in the event of two conflicting change withdrawal credentials. It also establishes a list of change withdrawal address signed messages to help broadcast "as soon as possible" when the network supports it, and encourage client teams to help use these lists to honour filter and prioritize accepting requests by REST and transmitting them via P2P. This will not change consensus, but may help prevent propagating an attack where a withdrawal key has been knowingly or unknowingly compromised. 

# Proposal

## Change Withdrawal Address Rebroadcast Delay
Beacon node clients optionally implement a "change withdrawal address rebroadcast delay" that creates an optional delay in rebroadcasting change withdrawal addresses (suggested to default to 2000 seconds (>5 epochs), or set to 0 seconds for no delay, or set to -1 to strictly rebroadcast requests matching a Change Withdrawal Address Broadcast signature or Change Withdrawal Address Acceptance list). This setting will allow change withdrawal address requests time for peer replication of client accepted valid requests that are preferred by the community. This can prevent a "first to arrive" critical race condition for a conflicting change withdraw address. 

## Change Withdrawal Address Broadcast
Support a file of lines specifying "validator index,current withdrawal public key,proposed execution layer withdrawal address,consensus layer signature" which will instruct nodes to automatically submit a one-time change withdrawal address broadcast message for each valid line. This file will give all node operators an opportunity to ensure their valid change withdrawal address messages are broadcast and heard by nodes during the first epoch supporting the change withdrawal address operation. It will also instruct nodes to perpetually prefer accepting and repeating signatures matching the signature in the file, and reject accepting or rebroadcasting messages which do not match a signature for a given withdrawal credential. 

At the first epoch which supports the change withdrawal address operation, a client node should automatically broadcast via P2P each valid message found in this file. 

## Change Withdrawal Address Acceptance
Support a file in the format "withdrawal credentials, execution layer address" which allows clients to load, or packaged by default, a verifiable list matching the consensus layer withdrawal credentials and the original execution layer deposit address. 

While any withdrawal credential and withdrawal address can be supported, this list can be used to help enforce a deposit address is given preference in rebroadcasting, even if other clients do not support or have not loaded a Change Withdrawal Address Broadcast file. 

## Change Withdrawal Address Handling
Clients should first rely on a "Change Withdrawal Address Broadcast" file of verifiable signatures, then fallback to a "Change Withdrawal Address Acceptance" file intended to be loaded with all validator original deposit address information, and then fallback to accept a "first request" but delay in rebroadcasting it via P2P. All of this proposal is optional, but we encourage all client teams to include a copy of the uncontested verification file and enable it by default to protect the community. This optional protection will prove the user was both in control of the consensus layer and execution layer address, while making sure their intended change withdrawal address message is ready to broadcast as soon as the network supports it. 

If a node is presented with a change withdrawal address operation via the REST API or P2P:

A) Withdrawal credential found in "Change Withdrawal Address Broadcast" file:
  1. Signature Match: If a valid change withdrawal request signature is received for a withdrawal credential that matches the first signature found in the "Change Withdrawal Address Broadcast" file, accept it via REST API, rebroadcast it via P2P, and drop any pending “first preferred” if existing. 
  2. Signature Mismatch: If a valid change withdrawal request is received for a withdrawal credential that does not match the first signature found in the "Change Withdrawal Address Broadcast" file, reject it via REST API, and drop it to prevent rebroadcasting it via P2P.

B) Withdrawal credential not found in or no "Change Withdrawal Address Broadcast" file:
1. Matching withdraw credential and withdraw address in "Change Withdrawal Address Acceptance" file: If a valid change withdrawal address request is received for a withdrawal credential that matches the first found withdrawal address provided in the "Change Withdrawal Address Acceptance" file, accept it via REST API, rebroadcast it via P2P, and drop any pending “first preferred” if existing. 
2. Mismatching withdraw credential and withdraw address in "Change Withdrawal Address Acceptance" file: If a valid change withdrawal request is received for a withdrawal credential that does not match the first found withdrawal address provided in the "withdrawal address" file, reject it via REST API, and drop it to prevent rebroadcasting it via P2P.
3. Missing withdraw address in or no "Change Withdrawal Address Acceptance" file: 

    i. First Preferred: If first valid change withdrawal request is received for a not finalized withdrawal credential that does not have any listed withdrawal credential entry in the "Change Withdrawal Address Acceptance" file, accept it via REST API, but do not yet rebroadcast it via P2P (“grace period”). Once the client “Change Withdrawal Address Grace Period” has expired and no other messages have invalidated this message, rebroadcast the request via P2P. 
  
    ii. Subsequent Rejected: If an existing valid "First Preferred" request exists for a not finalized withdrawal credential, reject it via REST API, and drop it to prevent rebroadcasting via P2P. 


Note that these restrictions do not apply to withdrawal credential change operations in blocks. If an operation has been included on-chain it is by definition valid regardless of its contents.

It is critical to understand that this proposal is not a consensus change. Nothing in this proposal restricts the validity of withdrawal credential change operations within the protocol. It is a voluntary change by client teams to build this functionality in to their beacon nodes, and a voluntary change by node operators to accept any or all of the restrictions and broadcasting capabilities suggested by end users.

Because of the above, even if fully implemented, it will be down to chance as to which validators propose blocks, and which voluntary restrictions those validators’ beacon nodes are running. Node operators can do what they will to help the community prevent attacks on any compromised consensus layer keys, but there are no guarantees of success this will prevent a successful attack. 

## Scenarios
### 1: Attacker lacks EL deposit key, uncontested claim
- User A: Controls the CL keys and the EL key used for the deposit
- User B: Controls the CL keys, but does not control the EL key for the deposit

User A signs and submits a claim to the CLWP repository, clients load User A message into the "Change Withdrawal Address Broadcast" file. At the time of the first epoch support Change Withdrawal Address, many (not all) nodes begin to broadcast the message. User B also tries to submit a different but valid Change Withdrawal Address to an address that does not match the signature in the claim. This message is successfully received via REST API, but some (not all) nodes begin to silently drop this message as the signature does not match the signature in the "Change Withdrawal Address Broadcast" file. As such, these nodes do not replicate this message via P2P. The nodes which do not have a Change Withdrawal Address Broadcast file loaded may still impose a "Change Withdrawal Address Rebroadcast Delay" to keep listening (for about 5 epochs) to see if there are any conflicts to this message. This delay may give User A an advantage in beating User B to consensus, but there is no certainty as it will depend on chance which validator and nodes are involved. 

## 2: Attacker has both EL deposit key and CL keys, uncontested claim
- User A: Controls the CL key/mnemonic and the EL key used for the deposit, and submits a claim to move to a new address
- User B: Controls the CL and EL key/mnemonic used for the EL deposit, but fails to submit a claim

It is possible/likely that User A would notice that all their funds in the EL deposit address had been stolen. This may signal that their CL key is compromised as well, so they decide to pick a new address for the withdrawal. The story will play out the same as Scenario 1 as the claim is uncontested. 

## 3: Same as #2, but the attacker submits a contested claim
- User A: Controls the CL keys/mnemonic and the EL key used for the deposit, and submits a claim to move to a new address
- User B: Controls the CL keys/mnemonic and the EL key used for the deposit, and submits a claim to move to a new address

This is a contested claim and as such there is no way to prove who is in control using on chain data. Instead, either user may try to persuade the community they are the rightful owner (identity verification, social media, etc.) in an attempt to get node operators to load their contested claim into their "Change Withdrawal Address Broadcast" file. However, there is no way to fully prove it. 

## 4: A user has lost either their CL key and/or mnemonic (no withdrawal key)
- User A: Lacks the CL keys and mnemonic

There is no way to recover this scenario with this proposal as we cannot prove a user has lost their keys, and the mnemonic is required to generate the withdrawal key. 

## 5: End game - attacker
- User A: Controls EL and CL key/mnemonic, successfully achieves a change address withdrawal
- User B: Controls CL key, decides to attack

Upon noticing User A has submitted a successful change address withdrawal, User B may run a validator and attempt to get User A slashed

## 6: Compromised key, but not vulnerable to withdrawal
- User A: Controls EL and CL key/mnemonic, but has a vulnerability which leaks their CL key but NOT their CL mnemonic
- User B: Controls the CL key, but lacks the CL mnemonic

User A may generate the withdrawal key (requires the mnemonic). User B can attack User A by getting them slashed, but will be unable to generate the withdrawal key. 

## 7: Attacker loads a malicious Change Withdrawal Address Broadcast and Change Withdrawal Address Acceptance files into one or multiple nodes, User A submits claim
- User A: Submits a valid uncontested claim which is broadcast out as soon as possible by many nodes
- User B: Submits no claim, but broadcasts a valid malicious claim out through their Change Withdrawal Address Broadcast list, and blocks User A's claim from their node.

User B's claim will make it into many nodes, but when it hits nodes that have adopted User A's signature they will be dropped and not rebroadcast. Statistically, User B will have a harder time achieving consensus among the entire community, but it will be down to chance. 

## 8: Same as #7, but User A submits no claim

The attacker will statistically likely win as they will be first to have their message broadcast to many nodes and, unless User A submits a request exactly at the time of support, it is unlikely to be heard by enough nodes to gain consensus. All users are encouraged to submit claims for this reason because nobody can be certain their mnemonic has not been compromised until it is too late. 

### Second Order Effects
1. A user who participates in the "Change Withdrawal Address Broadcast" may cause the attacker to "give up" early and instead start to slash. For some users, the thought of getting slashed is preferrable to giving an adversary any funds.
2. The attacker may set up their own unverified list of their own Change Withdrawal Address Acceptance file and nodes adopting this list to break ties in their favour. 
3. The attacker may set up their own Change Withdrawal Address Broadcast to reject signatures not matching their attack. 

# Documentation 
## Change Withdrawal Address Acceptance File
A file intended to be preloaded with all consensus layer withdrawal credentials and verifiable execution layer deposit addresses. This file will be generated by a script and able to be independently verified by community members using the consensus and execution layers, and intended to be included by all clients, enabled by default. Client nodes are encouraged to enable packaging this independently verifiable list with the client software, and enable it by default to help further protect the community from unsuspected attacks. 

depositAddress.csv format (both fields required):
```withdrawal credential, withdrawal address```

Example depositAddress.csv:
```
000092c20062cee70389f1cb4fa566a2be5e2319ff43965db26dbaa3ce90b9df99,01c34eb7e3f34e54646d7cd140bb7c20a466b3e852
0000d66cf353931500a54cbd0bc59cbaac6690cb0932f42dc8afeddc88feeaad6f,01c34eb7e3f34e54646d7cd140bb7c20a466b3e852
0000d6b91fbbce0146739afb0f541d6c21e8c41e92b97874828f402597bf530ce4,01c34eb7e3f34e54646d7cd140bb7c20a466b3e852
000037ca9a1cf2223d8b9f81a14d4937fef94890ae4fcdfbba928a4dc2ff7fcf3b,01c34eb7e3f34e54646d7cd140bb7c20a466b3e852
0000344b6c73f71b11c56aba0d01b7d8ad83559f209d0a4101a515f6ad54c89771,01f19b1c91faacf8071bd4bb5ab99db0193809068e
```

## Change Withdrawal Address Broadcast Claim
A community collected and independently verifiable list of "Change Withdrawal Address Broadcasts" containing verifiable claims will be collected. Client teams and node operators may verify these claims independently and are suggested to include "Uncontested and Verified" claims enabled by default in their package. 

To make a verifiable claim, users must upload using their GitHub ID with the following contents to the [CLWP repository](https://github.com/benjaminchodroff/ConsensusLayerWithdrawalProtection) in a text file "claims/validatorIndex-gitHubUser.txt" such as "123456-myGitHubID.txt"

123456-myGitHubID.txt:
```
current_withdrawal_public_key=b03c5ea17b017cffd22b6031575c4453f20a4737393de16a626fb0a8b0655fe66472765720abed97e8022680204d3868
proposed_withdrawal_address=0108f2e9Ce74d5e787428d261E01b437dC579a5164
consensus_layer_withdrawal_signature=
execution_layer_deposit_signature=
execution_layer_withdrawal_signature=
email=noreply@ethereum.org
```

| Key | Value | 
| ----| ------|
| **current_withdrawal_public_key** | (Required) The "pubkey" field found in deposit_data json file matching the validator index| Required | Necessary for verification |
| **proposed_withdrawal_address** | (Required) The address in Ethereum you wish to authorize withdrawals to, prefaced by "01" without any "0x", such that an address "0x08f2e9Ce74d5e787428d261E01b437dC579a5164" turns into "0108f2e9Ce74d5e787428d261E01b437dC579a5164 |
| **consensus_layer_withdrawal_signature** | (Required) The verifiable signature generated by signing "validator_index,current_withdrawal_public_key,proposed_withdrawal_address" using the consensus layer private key | 
| execution_layer_deposit_signature | (Optional) The verifiable signature generated by signing "validator_index,current_withdrawal_public_key,proposed_withdrawal_address" using the execution layer deposit address private key | 
| execution_layer_withdrawal_signature | (Optional) The verifiable signature generated by signing "validator_index,current_withdrawal_public_key,proposed_withdrawal_address" using the execution layer proposed withdrawal address private key. This may be the same result as the "execution_layer_deposit_signature" if the user intends to withdraw to the same execution layer deposit address. |
| email | (Optional) Any actively monitored email address to notify if contested | 

## Claim Acceptance
In order for a submission to be merged into CLWP GitHub repository, the submission must have:
1. Valid filename in the format validatorIndex-githubUsername.txt
2. Valid validator index which is deposited, pending, or active on the consensus layer 
3. Matching GitHub username in file name to the user submitting the request
4. Verifiable consensus_layer_withdrawal_signature, and a verifiable execution_layer_deposit_signature and execution_layer_withdrawal_signature if included
5. All required fields in the file with no other content present

All merge requests that fail will be provided a reason from above which must be addressed prior to merge. Any future verifiable amendments to accepted claims must be proposed by the same GitHub user, or it will be treated as a contention.

## Change Withdrawal Address Broadcast
Anyone in the community will be able to generate the following verifiable files from the claims provided:
	A. UncontestedVerified - Community collected list of all verifiable uncontested change withdrawal address final requests (no conflicting withdrawal credentials allowed from different GitHub users)
	B. ContestedVerified - Community collected list of all contested verifiable change withdrawal address requests (will contain only verifiable but conflicting withdrawal credentials from different GitHub users)

A claim will be considered contested if a claim arrives where the verifiable consensus layer signatures differ between two or more GitHub submissions, where neither party has proven ownership of the execution layer deposit address. If a contested but verified "Change Withdrawal Address Broadcast" request arrives to the GitHub community, all parties will be notified via GitHub, forced into the ContestedVerified list, and may try to convince the wider community by providing any off chain evidence supporting their claim to then include their claim in nodes. All node operators are encouraged to load the UncontestedVerified signatures file as enabled, and optionally append only ContestedVerified signatures that they have been convinced are the rightful owner in a manner to further strengthen the community. 

The uncontested lists will be of the format:

UncontestedVerified-datetime.txt:
```
validator_index,current_withdrawal_public_key,proposed_withdrawal_address,consensus_layer_withdrawal_signature
```

The contested list will be of the format:

ContestedVerified-datetime.txt
```
validator_index,current_withdrawal_public_key,proposed_withdrawal_address,consensus_layer_withdrawal_signature,email
```


## Specification
The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Ethereum platforms (go-ethereum, parity, cpp-ethereum, ethereumj, ethereumjs, and [others](https://github.com/ethereum/wiki/wiki/Clients)).

## Rationale
The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages.

## Backwards Compatibility
All EIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The EIP must explain how the author proposes to deal with these incompatibilities. EIP submissions without a sufficient backwards compatibility treatise may be rejected outright.

## Test Cases
This proposal does not change consensus. However, client test cases will be added here once the proposal is formalized.

## Reference Implementation
An optional section that contains a reference/example implementation that people can use to assist in understanding or implementing this specification.  If the implementation is too large to reasonably be included inline, then consider adding it as one or more files in `../assets/eip-####/`.

## Security Considerations
All EIPs must contain a section that discusses the security implications/considerations relevant to the proposed change. Include information that might be important for security discussions, surfaces risks and can be used throughout the life cycle of the proposal. E.g. include security-relevant design decisions, concerns, important discussions, implementation-specific guidance and pitfalls, an outline of threats and risks and how they are being addressed. EIP submissions missing the "Security Considerations" section will be rejected. An EIP cannot proceed to status "Final" without a Security Considerations discussion deemed sufficient by the reviewers.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
	
# ConsensusLayerWithdrawalProtection

ConsensusLayerWithdrawalProtection



# Consensus Layer Withdrawal Protection
First proposed by [Jim McDonald](https://ethresear.ch/u/jgm/summary), with contributions from [Pietje Puk](https://ethresear.ch/u/pietjepuk/summary), [Benjamin Chodroff](https://ethresear.ch/u/benjaminchodroff/summary) 

Thanks to the [Ethereum Research](https://ethresear.ch/) team, [ETHStaker](https://ethstaker.cc/) solo staking community, and the hackers who encourage us to make Ethereum even better





