# UCAN Delegation Specification v1.0.0-rc.1



TODO
- Remove general motivation section, narrow to delegation
- Consider batch signatures for batch use cases
  - Cature as an invocation? Seems like overkill
  - Define `alg: "batch/RS256"`, `alg: "batch/EdDSA"`, etc? No
- Scope all to DIDs













## Editors

* [Brooklyn Zelenka], [Fission]

## Authors

* [Brooklyn Zelenka], [Fission]
* [Daniel Holmgren], [Bluesky]
* [Irakli Gozalishvili], [Protocol Labs]
* [Philipp Krüger], [Fission]

## Dependencies

* [UCAN]
* [DID]
* [JWT]

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119].

# 0 Abstract

UCAN Delegation is a component of the [UCAN] specification. This specification describes the semantics and serialization format for [UCAN] delegation between principals. It provides public-key verifiable, delegable, expressive, openly extensible [capabilities] by extending the familiar [JWT] structure. UCANs achieve public verifiability with chained certificates and [decentralized identifiers (DIDs)][DID]. Verifiable chain compression is enabled via [content addressing]. Being encoded with the familiar JWT, UCAN improves the familiarity and adoptability of schemes like [SPKI/SDSI][SPKI] for web and native application contexts.

# 1 Introduction

## 1.1 Motivation


# 2 Terminology












## 2.11 Time

Time takes on [multiple meanings][time definition] in systems representing facts or knowledge. The senses of the word "time" are given below.

### 2.11.1 Valid Time Range

The period of time that a capability is valid from and until.

### 2.11.2 Assertion Time

The moment at which a delegation was asserted. This MAY be captured via an `iat` field, but is generally superfluous to capture in the token. "Assertion time" is useful when discussing the lifecycle of a token.

### 2.11.3 Decision (or Validation) Time

Decision time is the part of the lifecycle when "a decision" about the token is made. This is typically during validation, but also includes resolving external state (e.g. storage quotas).

# 3 JWT Structure

UCANs MUST be formatted as [JWT]s, with additional keys as described in this document. The overall container of a header, claims, and signature remains. Please refer to [RFC 7519][JWT] for more on this format.

## 3.1 Header

The header MUST include all of the following fields:
| Field | Type     | Description                    | Required |
|-------|----------|--------------------------------|----------|
| `alg` | `String` | Signature algorithm            | Yes      |
| `typ` | `String` | Type (MUST be `"JWT"`)         | Yes      |

The header is a standard JWT header.

EdDSA, as applied to JOSE (including JWT), is described in [RFC 8037].

Note that the JWT `"alg": "none"` option MUST NOT be supported. The lack of signature prevents the issuer from being validatable.

### 3.1.1 Examples

```json
{
  "alg": "EdDSA",
  "typ": "JWT",
}
```

## 3.2 Payload

The payload MUST describe the authorization claims, who is involved, and its validity period.

| Field | Type                                      | Description                                            | Required |
|-------|-------------------------------------------|--------------------------------------------------------|----------|
| `ucv` | `String`                                  | UCAN Semantic Version (`1.0.0-rc.1`)                   | Yes      |
| `iss` | `String`                                  | Issuer DID (sender)                                    | Yes      |
| `aud` | `String`                                  | Audience DID (receiver)                                | Yes      |
| `nbf` | `Integer` (53-bits[^js-num-size])         | Not Before UTC Unix Timestamp in seconds (valid from)  | No       |
| `exp` | `Integer \| null` (53-bits[^js-num-size]) | Expiration UTC Unix Timestamp in seconds (valid until) | Yes      |
| `nnc` | `String`                                  | Nonce                                                  | Yes      |
| `fct` | `{String: Any}`                           | Facts (asserted, signed data)                          | No       |
| `cap` | `{URI: {Ability: Caveat}}`                | Capabilities                                           | Yes      |

### 3.2.1 Version

The `ucv` field sets the version of the UCAN specification used in the payload.

### 3.2.2 Principals

The `iss` and `aud` fields describe the token's principals. They are distinguished by having DIDs. These can be conceptualized as the sender and receiver of a postal letter. The token MUST be signed with the private key associated with the DID in the `iss` field. Implementations MUST include the [`did:key`] method, and MAY be augmented with [additional DID methods][DID].

The `iss` and `aud` fields MUST contain a single principal each.

If an issuer's DID has more than one key (e.g. [`did:ion`], [`did:3`]), the key used to sign the UCAN MUST be made explicit, using the [DID fragment] (the hash index) in the `iss` field. The `aud` field SHOULD NOT include a hash field, as this defeats the purpose of delegating to an identifier for multiple keys instead of an identity.

It is RECOMMENDED that the following `did:key` types be supported:

- [RSA][did:key RSA]
- [EdDSA][did:key EdDSA]
- [P-256 ECDSA][did:key ECDSA]

#### 3.2.2.1 Examples

```json
"aud": "did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp",
"iss": "did:key:zDnaerDaTF5BXEavCrfRZEk316dpbLsfPDZ3WJ5hRTPFU2169",
```

```json
"aud": "did:ion:EiCrsG_DLDmSKic1eaeJGDtUoC1dj8tj19nTRD9ODzAjaQ",
"iss": "did:pkh:eth:0xb9c5714089478a327f09197987f16f9e5d936e8a",
```

```json
"aud": "did:ion:EiCrsG_DLDmSKic1eaeJGDtUoC1dj8tj19nTRD9ODzAjaQ",
"iss": "did:ion:test:EiANCLg1uCmxUR4IUkpW8Y5_nuuXLbAEwonQd4q8pflTnw#key-1",
```

```json
"aud": "did:ion:EiCrsG_DLDmSKic1eaeJGDtUoC1dj8tj19nTRD9ODzAjaQ",
"iss": "did:3:bafyreiffkeeq4wq2htejqla2is5ognligi4lvjhwrpqpl2kazjdoecmugi#yh27jTt7Ny2Pwdy",
```

### 3.2.3 Time Bounds

`nbf` and `exp` stand for "not before" and "expires at," respectively. These are standard fields from [RFC 7519][JWT] (JWT) (which in turn uses [RFC 3339]), and MUST represent seconds since the Unix epoch in UTC, without time zone or other offset. Taken together, they represent the time bounds for a token. These timestamps MUST be represented as the number of integer seconds since the Unix epoch. Due to limitations[^js-num-size] in numerics for certain common languages, timestamps outside of the range from $-2^{53} – 1$ to $2^{53} – 1$ MUST be rejected as invalid.

The `nbf` field is OPTIONAL. When omitted, the token MUST be treated as valid beginning from the Unix epoch. Setting the `nbf` field to a time in the future MUST delay using a UCAN. For example, pre-provisioning access to conference materials ahead of time but not allowing access until the day it starts is achievable with judicious use of `nbf`.

The `exp` field MUST be set. Following the [principle of least authority][POLA], it is RECOMMENDED to give a timestamp expiry for UCANs. If the token explicitly never expires, the `exp` field MUST be set to `null`. If the time is in the past at validation time, the token MUST be treated as expired and invalid.

Keeping the window of validity as short as possible is RECOMMENDED. Limiting the time range can mitigate the risk of a malicious user abusing a UCAN. However, this is situationally dependent. It may be desirable to limit the frequency of forced reauthorizations for trusted devices. Due to clock drift, time bounds SHOULD NOT be considered exact. A buffer of ±60 seconds is RECOMMENDED.

[^js-num-size]: JavaScript has a single numeric type ([`Number`][JS Number]) for both integers and floats. This representation is defined as a [IEEE-754] double-precision floating point number, which has a 53-bit significand.

#### 3.2.3.1 Example

```js
{
  "nbf": 1529496683,
  "exp": 1575606941,
  // ...
}
```

### 3.2.4 Nonce

The REQUIRED nonce parameter `nnc` MAY be any value. A randomly generated string is RECOMMENDED to provide a unique UCAN, though it MAY also be a monotonically increasing count of the number of links in the hash chain. This field helps prevent replay attacks and ensures a unique CID per delegation. The `iss`, `aud`, and `exp` fields together will often ensure that UCANs are unique, but adding the nonce ensures this uniqueness.

The recommeneded size of the nonce differs by key type. In many cases, a random 12-byte nonce is sufficient. If uncertain, check the nonce in your DID's cryptosuite.

This field SHOULD NOT be used to sign arbitrary data, such as signature challenges. See the [`fct`][Facts] field for more.

#### 3.2.4.1 Examples

``` json
{
  // ...
  "nnc": "1701-D"
}
```

### 3.2.5 Facts

The OPTIONAL `fct` field contains a map of arbitrary facts and proofs of knowledge. The enclosed data MUST be self-evident and externally verifiable. It MAY include information such as hash preimages, server challenges, a Merkle proof, dictionary data, etc.

#### 3.2.5.1 Examples

``` json
{
  "fct": {
    "challenges": {
      "example.com": "abcdef",
      "another.example.net": "12345"
    },
    "sha3_256": {
      "B94D27B9934D3E08A52E52D7DA7DABFAC484EFE37A5380EE9088F7ACE2EFCDE9": "hello world"
    }
  }
}
```

### 3.2.6 Capabilities & Attenuation

Capabilities MUST be presented as a map. This map is REQUIRED but MAY be empty.

This map MUST contain some or none of the following:
1. A strict subset (attenuation) of the capability authority from the next direct `prf` field
2. Capabilities composed from multiple proofs (see [rights amplification])
3. Capabilities originated by the `iss` DID (i.e. by parenthood)

The anatomy of a capability MUST be given as a mapping of resource URI to abilities to array of caveats.

```
{ $SUBJECT: { $ABILITY: [ ...$CAVEATS ] } }
```

#### 3.2.6.2 Abilities

Abilities MUST be presented as a string. By convention, abilities SHOULD be namespaced with a slash, such as `msg/send`. One or more abilities MUST be given for each resource.

#### 3.2.6.3 Caveats


FIXME add _so much_ clarification
- Semantics defined by resource & OG issuer
- 



 
<!--

FIXME push resource into caveats


#### 3.2.6.1 Resource

Resources MUST be unique and given as [URI]s.

Resources in the capabilities map MAY overlap. For example, the following MAY coexist as top-level keys in a capabilities map:

```json
"https://example.com",
"https://example.com/blog"
```
-->




Caveats MAY be open ended. Caveats MUST be understood by the executor of the eventual [invocation]. Caveats MUST prevent invocation otherwise. Caveats MUST be formatted as objects.

On validation, the caveat array MUST be treated as a logically disjunct (an "OR", NOT an "and"). In other words: passing validation against _any_ caveat in the array MUST pass the check. For example, consider the following capabilities:




``` json
{
  "did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp": "ucan/*",
  "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK": {
    "crud/create": {
      "uri": "dns:example.com",
      "record": "TXT"
    }
  },
  "did:web:example.com": {
    "crud/read": {
      "uri": "https://blog.example.com",
      "status": "published"
    },
    "crud/update": [
      {
        "uri": "https://example.com/newsletter/", 
        "status": "draft"
      },
      [
        {
          "uri": "https://blog.example.com",
          "status": "published"
          "tag": "news",
        }
        { "tag": "breaking" },
      ]
    ]
  }
}
```

The above MUST be interpreted as the set of capabilities below. If _any_ are matched, the check MUST pass validation.

| Subject                                                    | Ability       | Caveat                                                                                                 |
|------------------------------------------------------------|---------------|--------------------------------------------------------------------------------------------------------|
| `did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp` | `ucan/*`      | Always                                                                                                 |
| `did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK` | `crud/create` | `TXT` records on `dns:example.com`                                                                     |
| `did:web:example.com`                                      | `crud/read`   | Posts at `https://blog.example.com` with a `published` status                                          |
| `did:web:example.com`                                      | `crud/update` | Posts at `https://example.com/newsletter/` with the `draft` status                                     |
| `did:web:example.com`                                      | `crud/update` | Posts at `https://blog.example.com` with the `published` status and tagged with `published` and `news` |


### XXXX.XXXXX Normal Form

Note that all caveats need to be understable to th eexecitor

All of the above can be validly expressed in [DNF].

``` json
{
  "did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp": {
    "ucan/*": [[{}]]
  },
  "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK": {
    "crud/create": [ FIXME: ambiguity if you only pass crud/create, then later they add a resource. You want to know *what* you can update
      [
        { "uri": "dns:example.com" },
        { "record": "TXT" }
      ]
    ]
  },
  "did:web:example.com": {
    "crud/read": [
      [
        { "uri": "https://blog.example.com" },
        { "status": "published" }
      ]
    ],
    "crud/update": [
      [
        { "uri": "https://example.com/newsletter/" }, 
        { "status": "draft" }
      ],
      [
        { "uri": "https://blog.example.com" },
        { "status": "published" },
        { "tag": "news" },
        { "tag": "breaking" }
      ]
    ]
  }
}
```











The caveat array SHOULD NOT be empty, as an empty array means "in no case" (which is equivalent to not listing the ability). This follows from the rule that delegations MUST be of equal or lesser scope. When an array is given, an attenuated caveat MUST (syntactically) include all of the fields of the relevant proof caveat, plus the newly introduced caveats.

| Proof Caveats | Delegated Caveats | Is Valid? | Comment                                |
|---------------|-------------------|-----------|----------------------------------------|
| `[{}]`        | `[{}]`            | Yes       | Equal                                  |
| `[x]`         | `[x]`             | Yes       | Equal                                  |
| `[x]`         | `[{}]`            | No        | Escalation to any                      |
| `[{}]`        | `[x]`             | Yes       | Attenuates the `{}` caveat to `x`      |
| `[x]`         | `[y]`             | No        | Escalation by using a different caveat |
| `[x, y]`      | `[x]`             | Yes       | Removes a capability                   |
| `[x, y]`      | `[x, (y + z)]`    | Yes       | Attenuates existing caveat             |
| `[x, y]`      | `[x, y, z]`       | No        | Escalation by adding new capability    |

Note that for consistency in this syntax, the empty array MUST be equivalent to disallowing the capability. Conversely, an empty object MUST be treated as "no caveats".

| Proof Caveats          | Comment                                                       |
|------------------------|---------------------------------------------------------------|
| `[]`                   | No capabilities                                               |
| `{}`, `[{}]`, `[[{}]]` | Full capabilities for this resource/ability pair (no caveats) |

### 3.2.7 Proof of Delegation

Attenuations MUST be satisfied by matching the attenuated [FIXME: in invocation] capability to a proof in the invocation's [`prf` array][invocation prf].

Proofs MUST be resolvable by the recipient. A proof MAY be left unresolvable if it is not used as support for the top-level UCAN's capability chain. The exact format MUST be defined in the relevant transport specification. Some examples of possible formats include: a JSON object payload delivered with the UCAN, a federated HTTP endpoint, a DHT, or a shared database.

# 4 Reserved Namespaces

## 4.1 `ucan`

The `ucan` resource and abilty namespace MUST be reserved. Implementation of the [`ucan-uri`] spec is RECOMMENDED.

## 4.2 "Top" Ability

The "top" (or "super user") ability MUST be denoted `*`. The top ability grants access to all other capabilities for the specified resource, across all possible namespaces. Top corresponds to an "all" matcher, whereas [delegation] corresponds to "any" in the UCAN chain. The top ability is useful when "linking" agents by delegating all access to resource(s). This is the most powerful ability, and as such it SHOULD be handled with care.

``` mermaid
%%{ init: { 'flowchart': { 'curve': 'linear' } } }%%

flowchart BT
  *

  msg/* --> *
  subgraph msgGraph [ ]
    msg/send --> msg/*
    msg/receive --> msg/*
  end

  crud/* --> *
  subgraph crudGraph [ ]
    crud/read --> crud/*
    crud/mutate --> crud/*

    subgraph mutationGraph [ ]
        crud/create --> crud/mutate
        crud/update --> crud/mutate
        crud/destroy --> crud/mutate
    end
  end

  ... --> *
```

### 4.2.1 Bottom

In concept there is a "bottom" ability ("none" or "void"), but it is not possible to represent in an ability. As it is merely the absence of any ability, it is not possible to construct a capability with a bottom ability.

# 5 Validation

Each capability has its own semantics, which needs to be interpretable by the target resource handler. Therefore, a validator MUST NOT reject all capabilities when only one is not understood.

If _any_ of the following criteria are not met, the UCAN MUST be considered invalid:

## 5.1 Time Bounds

A UCAN's time bounds MUST NOT be considered valid if the current system time is before the `nbf` field or after the `exp` field. This is called "ambient time validity."

All proofs MUST contain time bounds equal to or broader than the UCAN being delegated (i.e. delegation maintains or narrows the time bounds). If the proof expires before the delegated UCAN — or starts after it — the validator MUST treat the entire UCAN as invalid. Delegation inside of the time bound is called "timely delegation." These conditions MUST hold even if the current wall clock time is inside of incorrectly delegated bounds. 

A UCAN is valid inclusive from the `nbf` time and until the `exp` field. If the current time is outside of these bounds, the UCAN MUST be considered invalid. When setting these bounds, a delegator or invoker SHOULD account for expected clock drift. Use of time bounds this way is called "timely invocation."

``` js
// Pseudocode

const ensureTime = async (ucan) => {
  ensureTimeBounds(ucan)
  await Promise.all(ucan.prf.forEach(async (cid) => {
    const proof = await getUcan(cid)
    ensureProofNbf(ucan, proof)
    ensureProofExp(ucan, proof)
  }))
}

// Helpers

const ensureTimeBounds = (ucan) => {
  if (!!proof.nbf && ucan.exp !== null && ucan.nbf > ucan.exp) {
    throw new Error("Time violation: UCAN starts after expiry")
  }
}

const ensureProofNbf = (ucan, proof) => {
  if (!!proof.nbf && ucan.nbf < proof.nbf) {
    throw new Error("Time escelation: delegation starts before proof starts")
  }
}

const ensureProofExp = (ucan, proof) => {
  if (proof.exp !== null && ucan.exp > proof.exp) {
    throw new Error("Time escelation: delegation ends after proof ends")
  }
}
```

## 5.2 Principal Alignment

In delegation, the `aud` field of every proof MUST match the `iss` field of the UCAN being delegated to. This alignment MUST form a chain back to the originating principal for each resource. 

This calculation MUST NOT take into account [DID fragment]s. If present, fragments are only intended to clarify which of a DID's keys was used to sign a particular UCAN, not to limit which specific key is delegated between. Use `did:key` if delegation to a specific key is desired.

``` mermaid
flowchart RL
  owner[/Alice\] -. owns .-> resource[(Storage)]
  executor[/"Compute Service"\] --> del2Aud
  rootIss --> owner

  executor -. accesses .-> resource
  rootCap -. references .-> resource

  subgraph root [Root UCAN]
    rootIss(iss: Alice)
    rootAud(aud: Bob)
    rootCap("cap: (Storage, crud/*)")
  end

  subgraph del1 [Delegated UCAN]
    del1Iss(iss: Bob) --> rootAud
    del1Aud(aud: Carol)
    del1Cap("cap: (Storage, crud/*)") --> rootCap
  end

  subgraph del2 [Final UCAN]
    del2Iss(iss: Carol) --> del1Aud
    del2Aud(aud: Compute Service)
    del2Cap("cap: (Storage, crud/*)") --> del1Cap
  end
```

In the above diagram, Alice has some storage. This storage may exist in one location with a single source of truth, but to help build intuition this example is location independent: local versions and remote stored copies are eventually consistent, and there is no one "correct" copy. As such, we list the owner (Alice) directly on the resource.

Alice delegates access to Bob. Bob then redelegates to Carol. Carol invokes the UCAN as part of a REST request to a compute service. To do this, she MUST both provide proof that she has access (the UCAN chain), and MUST delegate access to the invoking compute service. The invoking service MUST check that the root issuer (Alice) is in fact the owner (typically the creator) of the resource. This MAY be listed directly on the resource, as it is here. Once the UCAN chain and root ownership are validated, the storage service performs the write.

### 5.2.1 Recipient Validation

An agent executing a capability MUST verify that the outermost `aud` field _matches its own DID._ The associated ability MUST NOT be performed if they do not match. Recipient validation is REQUIRED to prevent the misuse of UCANs in an unintended context.

The following UCAN fragment would be valid to invoke as `did:key:zH3C2AVvLMv6gmMNam3uVAjZpfkcJCwDwnZn6z3wXmqPV`. Any other agent MUST NOT accept this UCAN. For example, `did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp` MUST NOT run the ability associated with that capability.

``` js
{
  "aud": "did:key:zH3C2AVvLMv6gmMNam3uVAjZpfkcJCwDwnZn6z3wXmqPV",
  "iss": "did:key:zAKJP3f7BD6W4iWEQ9jwndVTCBq8ua2Utt8EEjJ6Vxsf",
  // ...
}
```

A good litmus test for invocation validity by a invoking agent is to check if they would be able to create a valid delegation for that capability.

### 5.2.2 Token Uniqueness

Each remote invocation MUST be a unique UCAN: for instance using a nonce (`nnc`) or simply a unique expiry. The recipient MUST validate that they have not received the top-level UCAN before. For implementation recommentations, please refer to the [replay attack prevention] section. 

## 5.3 Proof Chaining

Each capability MUST either be originated by the issuer (root capability, or "parenthood") or have one-or-more proofs in the `prf` field to attest that this issuer is authorized to use that capability ("introduction"). In the introduction case, this check MUST be recursively applied to its proofs until a root proof is found (i.e. issued by the resource owner).

Except for rights amplification (below), each capability delegation MUST have equal or narrower capabilities from its proofs. The time bounds MUST also be equal to or contained inside the time bounds of the proof's time bounds. This lowering of rights at each delegation is called "attenuation."





FIXME removed section, fix numbering

## 5.5 Content Identifiers

A UCAN token MUST be referenced as a [base32] [CIDv1]. [SHA2-256] is the RECOMMENDED hash algorithm.

The [`0x55` raw data][raw data multicodec] codec MUST be supported. If other codecs are used (such as [`0x0129` `dag-json` multicodec][dag-json multicodec]), the UCAN MUST be able to be interpreted as a valid JWT (including the signature).

The resolution of these addresses is left to the implementation and end-user, and MAY (non-exclusively) include the following: local store, a distributed hash table (DHT), gossip network, or RESTful service. Please refer to [token resolution] for more.

## 5.5.1 CID Canonicalization

A canonical CID can be important for some use cases, such as caching and [revocation]. A canonical CID MUST conform to the following:

* [CIDv1]
* [base32]
* [SHA2-256]
* [Raw data multicodec] (`0x55`)

# 6 Token Resolution

Token resolution is transport specific. The exact format is left to the relevant UCAN transport specification. At minimum, such a specification MUST define at least the following:

1. Request protocol
2. Response protocol
3. Collections format

Note that if an instance cannot dereference a CID at runtime, the UCAN MUST fail validation. This is consistent with the [constructive semantics] of UCAN.

# 9. Acknowledgments

Thank you to [Brendan O'Brien] for real-world feedback, technical collaboration, and implementing the first Golang UCAN library.

Many thanks to [Hugo Dias], [Mikael Rogers], and the entire DAG House team for the real world feedback, and finding inventive new use cases.

Thank you [Blaine Cook] for the real-world feedback, ideas on future features, and lessons from other auth standards.

Many thanks to [Brian Ginsburg] and [Steven Vandevelde] for their many copy edits, feedback from real world usage, maintenance of the TypeScript implementation, and tools such as [ucan.xyz].

Many thanks to [Christopher Joel] for his real-world feedback, raising many pragmatic considerations, and the Rust implementation and related crates.

Many thanks to [Christine Lemmer-Webber] for her handwritten(!) feedback on the design of UCAN, spearheading the [OCapN] initiative, and her related work on [ZCAP-LD].

Thanks to [Benjamin Goering] for the many community threads and connections to [W3C] standards.

Thanks to [Juan Caballero] for the numerous questions, clarifications, and general advice on putting together a comprehensible spec.

Thank you [Dan Finlay] for being sufficiently passionate about [OCAP] that we realized that capability systems had a real chance of adoption in an ACL-dominated world.

Thanks to the entire [SPKI WG][SPKI/SDSI] for their closely related pioneering work.

Many thanks to [Alan Karp] for sharing his vast experience with capability-based authorization, patterns, and many right words for us to search for.

We want to especially recognize [Mark Miller] for his numerous contributions to the field of distributed auth, programming languages, and computer security writ large.

<!-- Internal Links -->

[Token Uniqueness]: #622-token-uniqueness
[canonical collections]: #71-canonical-json-collection
[content identifiers]: #65-content-identifiers
[delegation]: #51-ucan-delegation
[invocation prf]: FIXME
[replay attack prevention]: #93-replay-attack-prevention
[revocation]: #66-revocation
[rights amplification]: #64-rights-amplification
[token resolution]: #8-token-resolution
[top ability]: #41-top

<!-- External Links -->

[Alan Karp]: https://github.com/alanhkarp
[Benjamin Goering]: https://github.com/gobengo
[Biscuit]: https://github.com/biscuit-auth/biscuit/
[Blaine Cook]: https://github.com/blaine
[Bluesky]: https://blueskyweb.xyz/
[Brendan O'Brien]: https://github.com/b5
[Brian Ginsburg]: https://github.com/bgins
[Brooklyn Zelenka]: https://github.com/expede 
[CACAO]: https://blog.ceramic.network/capability-based-data-security-on-ceramic/
[CIDv1]: https://docs.ipfs.io/concepts/content-addressing/#identifier-formats
[Canonical CID]: #651-cid-canonicalization
[Capability Myths Demolished]: https://srl.cs.jhu.edu/pubs/SRL2003-02.pdf
[Christine Lemmer-Webber]: https://github.com/cwebber
[Christopher Joel]: https://github.com/cdata
[DID fragment]: https://www.w3.org/TR/did-core/#fragment
[DID path]: https://www.w3.org/TR/did-core/#path
[DID subject]: https://www.w3.org/TR/did-core/#dfn-did-subjects
[DID]: https://www.w3.org/TR/did-core/
[Dan Finlay]: https://github.com/danfinlay
[Daniel Holmgren]: https://github.com/dholms
[ECDSA security]: https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm#Security
[FIDO]: https://fidoalliance.org/fido-authentication/
[Fission]: https://fission.codes
[Hugo Dias]: https://github.com/hugomrdias
[IEEE 754]: https://ieeexplore.ieee.org/document/8766229
[Irakli Gozalishvili]: https://github.com/Gozala
[JS Number]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number
[JWT]: https://datatracker.ietf.org/doc/html/rfc7519
[Juan Caballero]: https://github.com/bumblefudge
[Local-First Auth]: https://github.com/local-first-web/auth
[Macaroon]: https://storage.googleapis.com/pub-tools-public-publication-data/pdf/41892.pdf
[Mark Miller]: https://github.com/erights
[Mikael Rogers]: https://github.com/mikeal/
[OCAP]: http://erights.org/elib/capability/index.html
[OCapN]: https://github.com/ocapn/ocapn
[POLA]: https://en.wikipedia.org/wiki/Principle_of_least_privilege
[Philipp Krüger]: https://github.com/matheus23
[Protocol Labs]: https://protocol.ai/
[RFC 2119]: https://datatracker.ietf.org/doc/html/rfc2119
[RFC 3339]: https://www.rfc-editor.org/rfc/rfc3339
[RFC 8037]: https://datatracker.ietf.org/doc/html/rfc8037
[SHA2-256]: https://en.wikipedia.org/wiki/SHA-2
[SPKI/SDSI]: https://datatracker.ietf.org/wg/spki/about/
[SPKI]: https://theworld.com/~cme/html/spki.html
[Seitan token exchange]: https://book.keybase.io/docs/teams/seitan
[Steven Vandevelde]: https://github.com/icidasset
[UCAN Invocation]: https://github.com/ucan-wg/invocation
[UCAN Revocation]: https://github.com/ucan-wg/revocation
[URI]: https://www.rfc-editor.org/rfc/rfc3986
[Verifiable credentials]: https://www.w3.org/2017/vc/WG/
[W3C]: https://www.w3.org/
[ZCAP-LD]: https://w3c-ccg.github.io/zcap-spec/
[`did:3`]: https://github.com/ceramicnetwork/CIPs/blob/main/CIPs/cip-79.md
[`did:ion`]: https://github.com/decentralized-identity/ion
[`did:key`]: https://w3c-ccg.github.io/did-method-key/
[base32]: https://github.com/multiformats/multibase/blob/master/multibase.csv#L12
[browser api crypto key]: https://developer.mozilla.org/en-US/docs/Web/API/CryptoKey
[capabilities]: https://en.wikipedia.org/wiki/Object-capability_model
[caps as keys]: http://www.erights.org/elib/capability/duals/myths.html#caps-as-keys
[confinement]: http://www.erights.org/elib/capability/dist-confine.html
[constructive semantics]: https://en.wikipedia.org/wiki/Intuitionistic_logic
[content addressable storage]: https://en.wikipedia.org/wiki/Content-addressable_storage
[content addressing]: https://en.wikipedia.org/wiki/Content-addressable_storage
[dag-json multicodec]: https://github.com/multiformats/multicodec/blob/master/table.csv#L104
[did:key ECDSA]: https://w3c-ccg.github.io/did-method-key/#p-256
[did:key EdDSA]: https://w3c-ccg.github.io/did-method-key/#ed25519-x25519
[did:key RSA]: https://w3c-ccg.github.io/did-method-key/#rsa
[disjunction]: https://en.wikipedia.org/wiki/Logical_disjunction
[invocation]: https://github.com/ucan-wg/invocation
[raw data multicodec]: https://github.com/multiformats/multicodec/blob/a03169371c0a4aec0083febc996c38c3846a0914/table.csv?plain=1#L41
[secure hardware enclave]: https://support.apple.com/en-ca/guide/security/sec59b0b31ff
[spki rfc]: https://www.rfc-editor.org/rfc/rfc2693.html
[time definition]: https://en.wikipedia.org/wiki/Temporal_database
[ucan-uri]: https://github.com/ucan-wg/ucan-uri
[ucan.xyz]: https://ucan.xyz













FIXME move to own spec?

# 7. Collections

FIXME update exmaple to latest format ...or break out into own spec :shrug:

UCANs are indexed by their hash — often called their ["content address"][content addressable storage]. UCANs MUST be addressable as [CIDv1]. Use of a [canonical CID] is RECOMMENDED.

Content addressing the proofs has multiple advantages over inlining tokens, including:
* Avoids re-encoding deeply nested proofs as Base64 many times (and the associated size increase)
* Canonical signature
* Enables only transmitting the relevant proofs 

Multiple UCANs in a single request MAY be collected into one table. It is RECOMMENDED that these be indexed by CID. The [canonical JSON representation][canonical collections] (below) MUST be supported. Implementations MAY include more formats, for example to optimize for a particular transport. Transports MAY map their collection to this collection format.

### 7.1 Canonical JSON Collection

The canonical JSON representation is an key-value object, mapping UCAN content identifiers to their fully-encoded base64url strings. A root "entry point" (if one exists) MUST be indexed by the slash `/` character.

#### 7.1.1 Example


``` json
{
  "/": "eyJhbGciOiJFZERTQSIsInR5cCI6IkpXVCIsInVjdiI6IjAuOC4xIn0.eyJhdWQiOiJkaWQ6a2V5Ono2TWtmUWhMSEJTRk11UjdiUVhUUWVxZTVrWVVXNTFIcGZaZWF5bWd5MXprUDJqTSIsImF0dCI6W3sid2l0aCI6eyJzY2hlbWUiOiJ3bmZzIiwiaGllclBhcnQiOiIvL2RlbW91c2VyLmZpc3Npb24ubmFtZS9wdWJsaWMvcGhvdG9zLyJ9LCJjYW4iOnsibmFtZXNwYWNlIjoid25mcyIsInNlZ21lbnRzIjpbIk9WRVJXUklURSJdfX0seyJ3aXRoIjp7InNjaGVtZSI6InduZnMiLCJoaWVyUGFydCI6Ii8vZGVtb3VzZXIuZmlzc2lvbi5uYW1lL3B1YmxpYy9ub3Rlcy8ifSwiY2FuIjp7Im5hbWVzcGFjZSI6InduZnMiLCJzZWdtZW50cyI6WyJPVkVSV1JJVEUiXX19XSwiZXhwIjo5MjU2OTM5NTA1LCJpc3MiOiJkaWQ6a2V5Ono2TWtyNWFlZmluMUR6akc3TUJKM25zRkNzbnZIS0V2VGIyQzRZQUp3Ynh0MWpGUyIsInByZiI6WyJleUpoYkdjaU9pSkZaRVJUUVNJc0luUjVjQ0k2SWtwWFZDSXNJblZqZGlJNklqQXVPQzR4SW4wLmV5SmhkV1FpT2lKa2FXUTZhMlY1T25vMlRXdHlOV0ZsWm1sdU1VUjZha2MzVFVKS00yNXpSa056Ym5aSVMwVjJWR0l5UXpSWlFVcDNZbmgwTVdwR1V5SXNJbUYwZENJNlczc2lkMmwwYUNJNmV5SnpZMmhsYldVaU9pSjNibVp6SWl3aWFHbGxjbEJoY25RaU9pSXZMMlJsYlc5MWMyVnlMbVpwYzNOcGIyNHVibUZ0WlM5d2RXSnNhV012Y0dodmRHOXpMeUo5TENKallXNGlPbnNpYm1GdFpYTndZV05sSWpvaWQyNW1jeUlzSW5ObFoyMWxiblJ6SWpwYklrOVdSVkpYVWtsVVJTSmRmWDFkTENKbGVIQWlPamt5TlRZNU16azFNRFVzSW1semN5STZJbVJwWkRwclpYazZlalpOYTJ0WGIzRTJVek4wY1ZKWGNXdFNibmxOWkZobWNuTTFORGxGWm5VMmNVTjFOSFZxUkdaTlkycEdVRXBTSWl3aWNISm1JanBiWFgwLlNqS2FIR18yQ2UwcGp1TkY1T0QtYjZqb04xU0lKTXBqS2pqbDRKRTYxX3VwT3J0dktvRFFTeFo3V2VZVkFJQVREbDhFbWNPS2o5T3FPU3cwVmc4VkNBIiwiZXlKaGJHY2lPaUpGWkVSVFFTSXNJblI1Y0NJNklrcFhWQ0lzSW5WamRpSTZJakF1T0M0eEluMC5leUpoZFdRaU9pSmthV1E2YTJWNU9ubzJUV3R5TldGbFptbHVNVVI2YWtjM1RVSktNMjV6UmtOemJuWklTMFYyVkdJeVF6UlpRVXAzWW5oME1XcEdVeUlzSW1GMGRDSTZXM3NpZDJsMGFDSTZleUp6WTJobGJXVWlPaUozYm1aeklpd2lhR2xsY2xCaGNuUWlPaUl2TDJSbGJXOTFjMlZ5TG1acGMzTnBiMjR1Ym1GdFpTOXdkV0pzYVdNdmNHaHZkRzl6THlKOUxDSmpZVzRpT25zaWJtRnRaWE53WVdObElqb2lkMjVtY3lJc0luTmxaMjFsYm5SeklqcGJJazlXUlZKWFVrbFVSU0pkZlgxZExDSmxlSEFpT2preU5UWTVNemsxTURVc0ltbHpjeUk2SW1ScFpEcHJaWGs2ZWpaTmEydFhiM0UyVXpOMGNWSlhjV3RTYm5sTlpGaG1jbk0xTkRsRlpuVTJjVU4xTkhWcVJHWk5ZMnBHVUVwU0lpd2ljSEptSWpwYlhYMC5TakthSEdfMkNlMHBqdU5GNU9ELWI2am9OMVNJSk1waktqamw0SkU2MV91cE9ydHZLb0RRU3haN1dlWVZBSUFURGw4RW1jT0tqOU9xT1N3MFZnOFZDQSJdfQ.Ab-xfYRoqYEHuo-252MKXDSiOZkLD-h1gHt8gKBP0AVdJZ6Jruv49TLZOvgWy9QkCpiwKUeGVbHodKcVx-azCQ",
  "bafkreihogico5an3e2xy3fykalfwxxry7itbhfcgq6f47sif6d7w6uk2ze": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsInVhdiI6IjAuMS4wIn0.eyJhdWQiOiJkaWQ6a2V5OnpTdEVacHpTTXRUdDlrMnZzemd2Q3dGNGZMUVFTeUExNVc1QVE0ejNBUjZCeDRlRko1Y3JKRmJ1R3hLbWJtYTQiLCJpc3MiOiJkaWQ6a2V5Ono1QzRmdVAyRERKQ2hoTUJDd0FrcFlVTXVKWmROV1dINU5lWWpVeVk4YnRZZnpEaDNhSHdUNXBpY0hyOVR0anEiLCJuYmYiOjE1ODg3MTM2MjIsImV4cCI6MTU4OTAwMDAwMCwic2NwIjoiLyIsInB0YyI6IkFQUEVORCIsInByZiI6bnVsbH0.Ay8C5ajYWHxtD8y0msla5IJ8VFffTHgVq448Hlr818JtNaTUzNIwFiuutEMECGTy69hV9Xu9bxGxTe0TpC7AzV34p0wSFax075mC3w9JYB8yqck_MEBg_dZ1xlJCfDve60AHseKPtbr2emp6hZVfTpQGZzusstimAxyYPrQUWv9wqTFmin0Ls-loAWamleUZoE1Tarlp_0h9SeV614RfRTC0e3x_VP9Ra_84JhJHZ7kiLf44TnyPl_9AbzuMdDwCvu-zXjd_jMlDyYcuwamJ15XqrgykLOm0WTREgr_sNLVciXBXd6EQ-Zh2L7hd38noJm1P_MIr9_EDRWAhoRLXPQ"
}
```

