# UCAN Delegation Specification v1.0.0-rc.1

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

UCAN Delegation is a component of [UCAN]. This specification describes the semantics and serialization format for [UCAN] delegation between principals. UCAN Delegation provides a cryptographically verifiable container, batched capabilties, heirarchical authority, and extensible caveats.

# 1 Introduction

## 1.1 Motivation

Design goals:

- Flexible pathing
- Extensibilty
- Consistency for interop

# 2 Token Structure

Regardless of how a Delegation is serialized on the wire, the sigature of a UCAN Delegation MUST be over a payload serialized as a [JWT] .

The overall container of a header, claims, and signature remain as per [RFC 7519][JWT].

## 3.1 Header

The header is a standard JWT header, and MUST include all of the following fields:

| Field | Type     | Description            | Required |
|-------|----------|------------------------|----------|
| `alg` | `String` | Signature algorithm    | Yes      |
| `typ` | `String` | Type (MUST be `"JWT"`) | Yes      |

### 3.1.1 Algorithms

It is RECOMMENDED that the following algorithms be supported:

- [RS256]
- [EdDSA][RFC 8037]
- [P-256 ECDSA]

All algorithms MUST match the DID principal in the `iss` field. This enforces that the `alg` field MUST be asymmetric (public key cryptography or nonstandard but emerging patterns like smart contract signatures)

The JWT `"alg": "none"` MUST NOT be supported, as the lack of signature prevents the issuer DID from being validated.

Symmetric algorithms such as HMACs (e.g. `"alg": "HS256"`) MUST NOT be supported, since they cannot be used to prove control over the issuer DID.

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

Note that every [Subject] MUST correspond to a root delegation issuer.

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

Several named points of time in the UCAN lifecycle [can be found in the high level spec].

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

#### 3.2.4.1 Example

``` json
{
  // ...
  "nnc": "NCC-1701-D"
}
```

### 3.2.5 Facts

FIXME not validated in the chain: do what you want!

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

# 4. Capabilities

Capabilities are the semantically-relevant claims of a delegation. They MUST be presented as a map under the `cap` field as a map. This map is REQUIRED but MAY be empty. This MUST take the following form:

```
{ $SUBJECT: { $ABILITY: $CAVEATS } }
```

This map MUST contain some or none of the following:
1. A strict subset (attenuation) of the capability authority from the next direct `prf` field
2. Capabilities composed from multiple proofs (see [rights amplification])
3. Capabilities originated by the `iss` DID (i.e. by parenthood)

The anatomy of a capability MUST be given as a mapping of resource URI to abilities to array of caveats.

Here is an illustrative example:

``` js
{
  // ...
  "cap": {
    "did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp": {
      "crud/create": {
        "uri": "https://blog.example.com/blog/",
        "status": "draft"
      },
      "msg/send": {
        "sender": "mailto:alice@example.com",
        "subject": "Weekly Reports",
        "day": "friday"
      }
    }
  }
}
```

## 4.1 Subject

The Subject MUST be the DID that initiated the delegation chain.

For example:

``` js
{
  // ...
  cap: {
    "did:web:example.com": "ucan/*"
  //└─────────┬─────────┘
  //       Subject
  }
}
```

## 4.2 Resource

Unlike Subjects and Abilities, Resources are semantic rather than syntactic. The Resource is the "what" that a capability describes.

By default, the Resource of a capability is the Subject. This makes the delegation chain self-certifying.

``` js
{
  // ...
  "cap": {
  //                    Subject & Resource
  //┌───────────────────────────┴────────────────────────────┐
    "did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp": {
      "crud/update": {
        "status": "draft"
      }
    }
  }
}
```

In the case where access to an [external resource] is delegated, the Subject MUST own the relationship to the Resource. The Resource SHOULD be referenced by a `uri` key in the relevant [Caveat](s), except where it would be clearer to do otherwise. This MUST be defined by the Subject and understood by the executor.

``` js
{
  // ...
  "cap": {
  //                         Subject
  //┌───────────────────────────┴────────────────────────────┐
    "did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp": {
      "crud/create": {
        "uri": "https://blog.example.com/blog/",
        //     └──────────────┬───────────────┘
        //                Resource
        "status": "draft"
      },
      "msg/send": {
        "sender": "mailto:alice@example.com",
        //        └───────────┬────────────┘
        //                Resource
        "subject": "Weekly Reports",
        "day": "friday"
      }
    }
  }
}
```

## 4.3 Abilities

Abilities MUST be presented as a string. By convention, abilities SHOULD be namespaced with a slash, such as `msg/send`. One or more abilities MUST be given for each resource.

``` js
{
  // ...
  "cap": {
    "did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp": {
  //     Ability
  //  ┌─────┴─────┐
      "crud/create": {
        "uri": "https://blog.example.com/blog/",
        "status": "draft"
      }
    }
  }
}
```

Abilities MAY be organized in a hierarchy that abstract over many [Operation]s. A typical example is a superuser capability ("anything") on a file system. Another is read vs write access, such that in an HTTP context, `WRITE` implies `PUT`, `PATCH`, `DELETE`, and so on. [`*` gives full access] to a Resource more in the vein of object capabilities. Organizing abilities this way allows Resources to evolve over time in a backward-compatible manner, avoiding the need to reissue UCANs with new resource semantics.

### 4.3.1 Reserved Abilities

#### 43.1.1 `ucan`

The `ucan` abilty namespace MUST be reserved. Support for `ucan/*` is RECOMMENDED

FIXME

#### 4.3.1.2 "Top" Ability

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

#### 4.3.1.3 Bottom

In concept there is a "bottom" ability ("none" or "void"), but it is not possible to represent in an ability. As it is merely the absence of any ability, it is not possible to construct a capability with a bottom ability.

## 4.4 Caveats

Caveats define a set of constraints on what can be redelegated or invoked. Caveat semantics MUST be established by the Subject. They are openly extensivle, but vocabularies may be reused across many Subjects.

Caveats MAY be open ended. Caveats MUST be understood by the executor of the eventual [invocation]. Caveats MUST be formatted as maps.

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

### 4.4.1 Normal Form

Caveats MAY be expressed in a compact form, but any caveat MUST be expressable in disjunctive normal form ([DNF]). Expanding to normal form during validation is RECOMMENDED to ease checking.

Normal form MUST take the following shape: `[[{}]]`. The outer array represents a logical `any` (chained `OR`s), and the inner arrays represent logical `all` (chained `AND`s).

For instance, the following represents `({a: 1, b:2} AND {c: 3}) OR {d: 4}`:

``` js
[
  [
    {        // ┐
      a: 1,  // ├─ Caveat ┐
      b: 2   // │         │
    },       // ┘         ├─ AND ┐
    {        // ┐         │      │
      c: 3   // ├─ Caveat ┘      │
    }        // ┘                ├─ OR
  ],         //                  │
  [          //                  │
    {d: 4}   // ]─ Caveat ───────┘
  ]
]
```







| Subject                                                    | Ability       | Caveat                                                                                                 |
|------------------------------------------------------------|---------------|--------------------------------------------------------------------------------------------------------|
| `did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp` | `ucan/*`      | Always                                                                                                 |
| `did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK` | `crud/create` | `TXT` records on `dns:example.com`                                                                     |
| `did:web:example.com`                                      | `crud/read`   | Posts at `https://blog.example.com` with a `published` status                                          |
| `did:web:example.com`                                      | `crud/update` | Posts at `https://example.com/newsletter/` with the `draft` status                                     |
| `did:web:example.com`                                      | `crud/update` | Posts at `https://blog.example.com` with the `published` status and tagged with `published` and `news` |


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

### 4.4.2 Compact Form





### 4.4.3 Top Caveat





# 5 Validation



FIXME note that validation happens at exection time

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

FIXME move to high level spec

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

## 5.4 Caveat Attenuation

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

<!--

FIXME move to invocatyion, maybe a small section here
### 3.2.7 Proof of Delegation

Attenuations MUST be satisfied by matching the attenuated [FIXME: in invocation] capability to a proof in the invocation's [`prf` array][invocation prf].

Proofs MUST be resolvable by the recipient. A proof MAY be left unresolvable if it is not used as support for the top-level UCAN's capability chain. The exact format MUST be defined in the relevant transport specification. Some examples of possible formats include: a JSON object payload delivered with the UCAN, a federated HTTP endpoint, a DHT, or a shared database.
 
 -->


# 6. Content Identifiers

A UCAN token MUST be referenced as a [base32] [CIDv1]. [SHA2-256] is the RECOMMENDED hash algorithm.

The [`0x55` raw data][raw data multicodec] codec MUST be supported. If other codecs are used (such as [`0x0129` `dag-json` multicodec][dag-json multicodec]), the UCAN MUST be able to be interpreted as a valid JWT (including the signature).

The resolution of these addresses is left to the implementation and end-user, and MAY (non-exclusively) include the following: local store, a distributed hash table (DHT), gossip network, or RESTful service. Please refer to [token resolution] for more.

## 5.5.1 CID Canonicalization

A canonical CID can be important for some use cases, such as caching and [revocation]. A canonical CID MUST conform to the following:

* [CIDv1]
* [base32]
* [SHA2-256]
* [Raw data multicodec] (`0x55`)


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


<!-- External Links -->

[external resource]: FIXME /spec sectioj 5.5 diagram
