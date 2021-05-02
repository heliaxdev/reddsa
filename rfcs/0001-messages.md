# FROST messages

Proposes a message layout to exchange information between participants of a FROST setup.

## Motivation

Currently FROST library is complete for 2 round signatures with a dealer/aggregator setup.
This proposal is only considering that specific features, additions and upgrades will need to be made when DKG is implemented.

Assuming all participants have a FROST library available we need to define message structures in a way that data can be exchanged between participants. The proposal is a collection of data types so each side can do all the actions needed for a real life situation.

## Definitions

- `dealer`
- `aggergator`
- `signer`
- `nonce`
- `commitment`
- 

## Guide-level explanation

We propose a message separated in 2 parts, a header and a payload:

```rust
struct Message {
    header: Header,
    payload: Payload,
}
```

`Header` will look as follows:

```rust
struct Header {
    msg_type: MsgType,
    version: MsgVersion,
    sender: Participant,
    receiver: Participant, 
}
```

While `Payload` will be defined as:

```rust
enum Payload {
    DealerBroadcast(MsgDealerBroadcast),
    Commitments(MsgCommitments),
    SigningPackage(MsgSigningPackage),
    SignatureShare(MsgSignatureShare),
    FinalSignature(MsgFinalSignature),
}
```

All the messages and new types will be defined in a new file `src/frost/messages.rs`

## Reference-level explanation

Here we explore in detail the header types and all the message payloads. 

### Header

Fields of the header define new types. Proposed implementation for them is as follows:

```rust
enum MsgType {
    DealerBroadcast = 1,
    Commitments = 2,
    SigningPackage = 3,
    SignatureShare = 4,
    FinalSignature = 5,
}

enum MsgVersion {
    FirstVersion = 1,
    ..
}

struct Participant(u8);
```

### Payloads

Each payload defines a new message:

```rust
// The dealer broadcast initial data, dealer must send
// this message to each participant involved.
struct MsgDealerBroadcast {
    share: frost::SharePackage,
}

// The signer participants sends the commitments to the
// aggregator.
struct MsgCommitments {
    commitment: frost::SigningCommitments,
}

// The aggergator decide what message to be signed, 
// prepare it and broadcast to signers.
struct MsgSigningPackage {
    signing_package: frost::SigningPackage,
}

// Each signer send the signatures to the agregator who is going to collect them 
// and generate a final spend signature 
struct MsgSignatureShare {
    signature_share: frost::SignatureShare,
}

// The final signature is broadcasted by the aggegator 
// to any participant.
struct MsgFinalSignature {
    final_signature: Signature<SpendAuth>,
}
```
## Serialization/Deserialization

Each message struct needs to serialize to bytes representation before it is sent through the wire and must deserialize to the same struct (round trip) on the receiver side. We use `serde` and macro derivations (`Serialize` and `Deserialize`) to automatically implement where possible.

This will require deriving serde in several types defined in `frost.rs`. 
Manual implementation of serialization/deserialization will be located at a new mod `src/frost/serialize.rs`.

FROST inherit types from `jubjub` such as `Scalar`, `ExtendedPoint`, `AffinePoint`, etc. We need to decide how serialization of these types that are defined in external crates will be done (maybe with wrappers?).

## Validation

Validation is implemented to each new data type as needed. This will ensure the creation of valid messages before they are sent. For example, in the header sender and receiver can't be the same:

```rust
impl Header {
    pub fn validate(&self) {
        if self.sender.0 == self.receiver.0 {
            panic!("sender and receiver are the same");
        }
    }
}
```

## Testing plan

- Create a happy path unit test similar to https://github.com/ZcashFoundation/redjubjub/blob/frost-messages/tests/frost.rs#L7 and:
  - Make messages on each step.
  - Simulate send/receive.
  - Test round trip serialization/deserialization on each message.
- Create property tests for each message.