# UCAN Delegation Specification v1.0.0-rc.1

## Editors

- [Brooklyn Zelenka], [Fission]

## Authors

- [Brooklyn Zelenka], [Fission]
- [Daniel Holmgren], [Bluesky]
- [Irakli Gozalishvili], [Protocol Labs]
- [Philipp Krüger], [Fission]

## Dependencies

- [DID]
- [IPLD]
- [UCAN]

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119].

# 0. Abstract

This specification describes the semantics and serialization format for [UCAN] delegation between principals. UCAN Delegation provides a cryptographically verifiable container, batched capabilities, hierarchical authority, and extensible caveats.
  
# 1. Introduction

UCAN Delegation is a certificate capability system with runtime-extensibility, ad hoc caveats, cacheability, and focused on ease of use and interoperability. Delegations act as a proofs for [UCAN Invocation]s.

Delegation provides a way to "transfer authority without transferring cryptographic keys". As an authorization system, it is more interested in "what" can be done than a list of "who can do what". For more on how Delegation fits into UCAN, please refer to the [high level spec][UCAN].

# 2. Delegation (Envelope)

Delegations MUST include a signature that validates against the `iss` DID. A Delegation Payload on its own MUST NOT be considered a valid Delegation.

| Field   | Type        | Required | Description                                  |
|---------|-------------|----------|----------------------------------------------|
| `pld`   | `&Payload`  | Yes      | The CID of the [Delegation Payload][Payload] |
| `sig`   | `Signature` | Yes      | The `iss`'s [Signature] over the `ucd` field |

# 3. Delegation Payload

The payload MUST describe the authorization claims, who is involved, and its validity period.

| Field  | Type                                      | Required | Description                                                 |
|--------|-------------------------------------------|----------|-------------------------------------------------------------|
| `udv`  | `String`                                  | Yes      | UCAN Semantic Version (`1.0.0-rc.1`)                        |
| `iss`  | `DID`                                     | Yes      | Issuer DID (sender)                                         |
| `aud`  | `DID`                                     | Yes      | Audience DID (receiver)                                     |
| `nbf`  | `Integer` (53-bits[^js-num-size])         | No       | "Not before" UTC Unix Timestamp in seconds (valid from)     |
| `exp`  | `Integer \| null` (53-bits[^js-num-size]) | Yes      | Expiration UTC Unix Timestamp in seconds (valid until)      |
| `nnc`  | `Bytes`                                   | Yes      | Nonce                                                       |
| `mta`  | `{String : Any}`                          | No       | [Meta] (asserted, signed data) — is not delegated authority |
| `sub`  | `DID`                                     | Yes      | Principal that the chain is about (the [Subject])           |
| `can`  | `String`                                  | Yes      | The [Command] to eventually invoke                          |
| `args` | `{String : Any}`                          | Yes      | Any [Arguments] that MUST be present in the Invocation      |
| `if`   | `[Caveat]`                                | Yes      | Any additional [Caveat]s                                    |

The payload MUST be serialized as [IPLD] and [signed over][Envelope]. The RECOMMENDED IPLD codec is [DAG-CBOR].

## 3.1 Version

The `udv` field sets the version of the UCAN Delegation specification used in the payload.

## 3.2 Principals

The `iss` and `aud` fields describe the token's principals. They are distinguished by having DIDs. These can be conceptualized as the sender and receiver of a postal letter. The token MUST be signed with the private key associated with the DID in the `iss` field. Implementations MUST include the [`did:key`] method, and MAY be augmented with [additional DID methods][DID].

The `iss` and `aud` fields MUST contain a single principal each.

If an issuer's DID has multiple or mutable keys (e.g. [`did:plc`], [`did:web`]), the key used to sign the UCAN MUST be made explicit, using the [DID fragment] (the hash index) in the `iss` field. The `aud` field SHOULD NOT include a hash field, as this defeats the purpose of delegating to an identifier for multiple keys instead of an identity.
 
[EdDSA] `did:key`s MUST be supported, and their use is RECOMMENDED. [RSA][did:key RSA] and [P-256 ECDSA][did:key ECDSA] `did:key`s MUST be supported, but SHOULD NOT be used when other options are available.

Note that every [Subject] MUST correspond to a root delegation issuer.

Below are a couple examples:

```json
"aud": "did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp",
"iss": "did:key:zDnaerDaTF5BXEavCrfRZEk316dpbLsfPDZ3WJ5hRTPFU2169",
```

```json
"aud": "did:web:example.com",
"iss": "did:pkh:eth:0xb9c5714089478a327f09197987f16f9e5d936e8a",
```

```json
"aud": "did:plc:ewvi7nxzyoun6zhxrhs64oiz",
"iss": "did:web:example.com",
```

```json
"aud": "did:pkh:eth:0xb9c5714089478a327f09197987f16f9e5d936e8a",
"iss": "did:plc:ewvi7nxzyoun6zhxrhs64oiz",
```

## 3.3 Time Bounds

`nbf` and `exp` stand for "not before" and "expires at," respectively. These MUST be expressed as seconds since the Unix epoch in UTC, without time zone or other offset. Taken together, they represent the time bounds for a token. These timestamps MUST be represented as the number of integer seconds since the Unix epoch. Due to limitations[^js-num-size] in numerics for certain common languages, timestamps outside of the range from $-2^{53} – 1$ to $2^{53} – 1$ MUST be rejected as invalid.

The `nbf` field is OPTIONAL. When omitted, the token MUST be treated as valid beginning from the Unix epoch. Setting the `nbf` field to a time in the future MUST delay using a UCAN. For example, pre-provisioning access to conference materials ahead of time but not allowing access until the day it starts is achievable with judicious use of `nbf`.

The `exp` field MUST be set. Following the [principle of least authority][PoLA], it is RECOMMENDED to give a timestamp expiry for UCANs. If the token explicitly never expires, the `exp` field MUST be set to `null`. If the time is in the past at validation time, the token MUST be treated as expired and invalid.

Keeping the window of validity as short as possible is RECOMMENDED. Limiting the time range can mitigate the risk of a malicious user abusing a UCAN. However, this is situationally dependent. It may be desirable to limit the frequency of forced reauthorizations for trusted devices. Due to clock drift, time bounds SHOULD NOT be considered exact. A buffer of ±60 seconds is RECOMMENDED.

Several named points of time in the UCAN lifecycle can be found in the [high level spec][UCAN].

Below are a couple examples:

```js
{
  // ...
  "nbf": 1529496683,
  "exp": 1575606941
}
```

```js
{
  // ...
  "exp": 1575606941
}
```

```js
{
  // ...
  "nbf": 1529496683,
  "exp": null
}
```

## 3.4 Nonce

The REQUIRED nonce parameter `nnc` MAY be any value. A randomly generated string is RECOMMENDED to provide a unique UCAN, though it MAY also be a monotonically increasing count of the number of links in the hash chain. This field helps prevent replay attacks and ensures a unique CID per delegation. The `iss`, `aud`, and `exp` fields together will often ensure that UCANs are unique, but adding the nonce ensures this uniqueness.

The recommended size of the nonce differs by key type. In many cases, a random 12-byte nonce is sufficient. If uncertain, check the nonce in your DID's crypto suite.

This field SHOULD NOT be used to sign arbitrary data, such as signature challenges. See the [`mta`][Meta] field for more.

Here is a simple example.

``` js
{
  // ...
  "nonce": {"/": {"bytes": "bGlnaHQgd29yay4"}}
}
```

## 3.5 Meta

The OPTIONAL `mta` field contains a map of arbitrary metadata, facts, and proofs of knowledge. The enclosed data MUST be self-evident and externally verifiable. It MAY include information such as hash preimages, server challenges, a Merkle proof, dictionary data, etc.

The data contained in this map MUST NOT be semantically meaningful to delegation chains.

Below is an example:

``` js
{
  // ...
  "meta": {
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

# 4. Capability

Capabilities are the semantically-relevant claims of a delegation. They MUST be presented as a map under the `cap` field as a map. This map is REQUIRED but MAY be empty. This MUST take the following form:

| Field  | Type               | Required | Description                                                                                          |
|--------|--------------------|----------|------------------------------------------------------------------------------------------------------|
| `sub`  | `URI`              | Yes      | The [Subject] that this Capability is about                                                          |
| `can`  | `Command`          | Yes      | The [Command] of this Capability                                                                     |
| `args` | `{String : Any}`   | Yes      | Any arguments that MUST be present _verbatim_ in the [Invocation]                                    |
| `if`   | `[{String : Any}]` | Yes      | Any additional caveats, such as regex matchers, contextual information (e.g. day of week), and so on |

Here is an illustrative example:

``` js
{
  // ...
  "sub": "did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp"
  "can": "blog/post/create",
  "args": {
    "tags": ["weekend", "news"],
    "status": "draft",
  },
  "if": [
    {
      "day": "friday"
    },
    {
      "field": "title",
      "match": "/^From the newsroom:/"
    },
    {
      "field": "body",
      "max-chars": 2000
    }
  ]
}
```

## 4.1 Subject

The Subject MUST be the DID that initiated the delegation chain.

For example:

``` js
{
  "sub": "did:web:example.com",
  // ...
}
```

## 4.2 Resource

Unlike Subjects and Commands, Resources are semantic rather than syntactic. The Resource is the "what" that a capability describes.

By default, the Resource of a capability is the Subject. This makes the delegation chain self-certifying.

``` js
{
  "sub": "did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp", // Subject & Resource
  "can": "crud/update",
  "args": {"status": "draft"}
  // ...
}
```

In the case where access to an [external resource] is delegated, the Subject MUST own the relationship to the Resource. The Resource SHOULD be referenced by a `uri` key in the relevant [Caveat(s)][Caveat], except where it would be clearer to do otherwise. This MUST be defined by the Subject and understood by the executor.

``` js
{
  "sub": "did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp",
  "can": "crud/create",
  "args":{
    "uri": "https://example.com/blog/", // Resource
    "status": "draft"
  },
  // ...
}
```

## 4.3 Command

Commands MUST be presented as a case-insensitive string. By convention, Commands SHOULD be namespaced with a slash, such as `msg/send`.

``` js
{
  "sub": "did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp": {
  "can": "msg/send",
  "arg": {
    "from": "mailto:alice@example.com",
    "to": "mailto:bob@example.com"
  },
  // ...
}
```

Abilities MAY be organized in a hierarchy that abstract over many [Command]s. A typical example is a superuser capability ("anything") on a file system. Another is mutation vs read access, such that in an HTTP context, `write` implies `put`, `patch`, `delete`, and so on. [`*` gives full access][Wildcard Ability] to a Resource more in the vein of object capabilities. Organizing abilities this way allows Resources to evolve over time in a backward-compatible manner, avoiding the need to reissue UCANs with new resource semantics.

### 4.3.1 Reserved Commands

#### 4.3.1.1 `ucan/` Namespace

The `ucan/` ability namespace MUST be reserved. This MUST include any ability string matching the regex `/^ucan\/.*/`

Support for the `ucan/*` delegated proof ability is RECOMMENDED.

#### 4.3.1.2 `*` AKA "Wildcard"

_"Wildcard" (`*`) is the most powerful ability, and as such it SHOULD be handled with care and used sparingly._

The "wildcard" (or "any", or "top") ability MUST be denoted `*`. This can be thought of as something akin to a super user permission in RBAC. 

The wildcard ability grants access to all other capabilities for the specified resource, across all possible namespaces. The wildcard ability is useful when "linking" agents by delegating all access to another device controlled by the same user, and that should behave as the same agent. It is extremely powerful, and should be used with care. Among other things, it permits the delegate to update a Subject's mutable DID document (change their private keys), revoke UCAN delegations, and use any resources delegated to the Subject by others.

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

Conceptually it has this shape:

``` ts
{
  "can": "*",
  "args": {
    "can": string, // Some other command
    "args": {[string]: any} // That commad's arguments
  },
  // ...
}
```

Since the nesting is fully redundant and infinitely nestable, it is instead used only in proof chains, and SHOULD NOT be invoked directly.

## 4.4 Arguments

The `args` field MAY contain partially applied Arguments for the shape of data specified by the [Command]. For example, below is an example that constrains sending email from a particular address:

``` js
// Delegation
{
  "can": "msg/send",
  "args": {
    "from": "alice@example.com",
    "subject": "Coffee"
  },
  // ...
}

// Valid invocation
{
  "do": "msg/send",
  "args": {
    "from": "alice@example.com", // Matches above
    "to": ["bob@example.com", "carol@elsewhere.example.com"],
    "subject": "Coffee",
    "body": "Still on for coffee" // Matches above
  },
  // ...
}
```

Any arguments MUST be taken verbatim and MUST NOT be further adjusted. For more flexible validation of Arguments, use [Caveats].

Note that this also applies to arrays and objects. For example, the `to` array in this example is considered to be exact, so the Invocation fails validation in this case:

``` js
// Delegation
{
  "can": "msg/send",
  "args": {
    "from": "alice@example.com",
    "to": ["bob@example.com"],
  },
  // ...
}

// INVALID Invocation
{
  "do": "msg/send",
  "args": {
    "from": "alice@example.com", // Matches above
    "to": ["bob@example.com", "carol@elsewhere.example.com"], // Does not match exactly
    "subject": "Coffee",
    "body": "Still on for coffee"
  },
  // ...
}
```

The indended logic is expressible with [Caveats].

## 4.5 Caveats

Caveats (the `if` field) constrain the capability in two ways:

- Syntactic constraints on [Arguments] (length, regex, inclusion)
- Environmental / contextual conditions (day of week)

Caveat semantics MUST be established by the Subject. They are openly extensible, but vocabularies may be reused across many Subjects. Caveats MUST be Understood by the [Executor] of the eventual [Invocation][UCAN Invocation]. Each caveat MUST be formatted as a map.

``` js
// Delegation
{
  "sub": "did:web:example.com",
  "can": "msg/send",
  "args": {
    "from": "alice@example.com"
  }
  "if": [
     {                               // ┐
       "day": "monday",              // ├─ Caveat
     },                              // ┘
     {                               // ┐
       "field": "to",                // ├─ Caveat
       "includes": "bob@exmaple.com" // │
     }                               // ┘
  ],
  // ...
}

// Valid Invocation, if Monday
{
  "do": "msg/send",
  "args": {
    "from": "alice@example.com",
    "to": ["bob@example.com", "carol@elsewhere.example.com"], // Matches criteria
    "subject": "Coffee",
    "body": "Still on for coffee"
  },
  // ...
}
```
  
The above Delegation MUST be interpreted as "may send email from `alice@example.com` on Mondays as long as `bob@exmaple.com` is among the recipients"

The `if` field MUST take the following shape: `[{}]`. The array represents a logical `all` (chained `AND`s). To represent logical `OR`, issue another delegation with that attenuation. For instance, the following represents `{a: 1, b:2} AND {c: 3} AND {d: 4}`:

``` js
[
  {       // ┐
    a: 1, // ├─ Caveat ─┐
    b: 2  // │          │
  },      // ┘          ├─ AND ─┐
  {       // ┐          │       │
    c: 3  // ├─ Caveat ─┘       │
  },      // ┘                  ├─ AND
  {       // ┐                  │
    d: 4  // ├─ Caveat ─────────┘
  }       // ┘
]
```

Expressing caveats in this standard way simplifies ad hoc extension at delegation time. As a concrete example, below a caveat is added to the constraints on the `to` field.

``` js
// Original Caveat
[
  {
    "field": "to",
    "includes": "bob@example.com"
  }
]

// Attenuated Caveat
[
  { // Same as above
    "field": "to",
    "includes": "bob@example.com"
  },
  { // Also must send to carol@example.com
    "field": "to",
    "includes": "carol@example.com"
  },
  { // Only send to @example.com addresses
    "field": "to",
    "elements": {
      "match": "/.+@example.com/" 
    }
  }
]
```

### 4.4.1 The Empty Caveat

The "empty" caveat MUST represent the lack of caveats. This is equivalent to `true` in predicate terms. It SHOULD be represented as `[]`, but `[{}]` MAY be used equivalently ("without any caveats").

``` js
{
  "if": [],
  // ...
}

{
  "if": [{}],
  // ...
}
```

# 5 Validation

Validation of a UCAN chain MAY occur at any time, but MUST occur upon receipt of an [Invocation] prior to execution. While proof chains exist outside of a particular delegation (and are made concrete in [UCAN Invocation]s), each delegate MUST store one or more valid delegations chains for a particular claim.

Each capability has its own semantics, which needs to be interpretable by the [Executor]. Therefore, a validator MUST NOT reject all capabilities when one that is not relevant to them is not understood. For example, if a caveat fails a delegation check at execution time, but is not relevant to the invocation, it MUST be ignored.

If _any_ of the following criteria are not met, the UCAN MUST be considered invalid:

## 5.1 Time Bounds

A UCAN's time bounds MUST NOT be considered valid if the current system time is before the `nbf` field or after the `exp` field. This is called the "validity period." Proofs in a chain MAY have different validity periods, but MUST all be valid at execution-time. This has the effect of making a delegation chain valid between the latest `nbf` and earliest `exp`.

``` js
// Pseudocode

const ensureTime = (delegationChain, now) => {
  delegationChain.forEach((ucan) => {
    if (!!ucan.nbf && now < can.nbf) {
      throw new Error(`Delegation is not yet valid, but will become valid at ${ucan.nbf}`)
    }

    if (ucan.exp !== null && now > ucan.exp) {
      throw new Error(`Delegation expired at ${ucan.exp}`)
    }
  })
}
```

## 5.2 Principal Alignment

In delegation, the `aud` field of every proof MUST match the `iss` field of the UCAN being delegated to. This alignment MUST form a chain back to the originating principal for each resource. 

This calculation MUST NOT take into account [DID fragment]s. If present, fragments are only intended to clarify which of a DID's keys was used to sign a particular UCAN, not to limit which specific key is delegated between. Use `did:key` if delegation to a specific key is desired.

``` mermaid
flowchart RL
    invoker((&nbsp&nbsp&nbsp&nbspDan&nbsp&nbsp&nbsp&nbsp))
    subject((&nbsp&nbsp&nbsp&nbspAlice&nbsp&nbsp&nbsp&nbsp))

    subject -- controls --> resource[(Storage)]
    rootCap -- references --> resource

    subgraph Delegations
        subgraph root [Root UCAN]
            subgraph rooting [Root Issuer]
                rootIss(iss: Alice)
                rootSub(sub: Alice)
            end

            rootCap("cap: (Storage, crud/*)")
            rootAud(aud: Bob)
        end

        subgraph del1 [Delegated UCAN]
            del1Iss(iss: Bob) --> rootAud
            del1Sub(sub: Alice)
            del1Aud(aud: Carol)
            del1Cap("cap: (Storage, crud/*)") --> rootCap


            del1Sub --> rootSub
        end

        subgraph del2 [Delegated UCAN]
            del2Iss(iss: Carol) --> del1Aud
            del2Sub(sub: Alice)
            del2Aud(aud: Dan)
            del2Cap("cap: (Storage, crud/*)") --> del1Cap

            del2Sub --> del1Sub
        end
    end

     subgraph inv [Invocation]
        invIss(iss: Dan)
        args("args: [Storage, crud/update, (key, value)]")
        invSub(sub: Alice)
        prf("proofs")
    end

    invIss --> del2Aud
    invoker --> invIss
    args --> del2Cap
    invSub --> del2Sub
    rootIss --> subject
    rootSub --> subject
    prf --> Delegations
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

## 5.3 Caveat Attenuation

The caveat array SHOULD NOT be empty, as an empty array means "in no case" (which is equivalent to not listing the ability). This follows from the rule that delegations MUST be of equal or lesser scope. When an array is given, an attenuated caveat MUST (syntactically) include all of the fields of the relevant proof caveat, plus the newly introduced caveats.

Here are some abstract examples:

| Proof Caveats        | Delegated Caveats  | Is Valid? | Comment                               |
|----------------------|--------------------|-----------|---------------------------------------|
| `[]`                 | `[]`               | Yes       | Equal                                 |
| `[]`                 | `[{}]`             | Yes       | [True Caveat]                         |
| `[{}]`               | `[]`               | Yes       | [True Caveat]                         |
| `[{}]`               | `[{}]`             | Yes       | Equal                                 |
| `[{a: 1}]`           | `[{a: 1}]`         | Yes       | Equal                                 |
| `[{}]`               | `[{a: 1}]`         | Yes       | Attenuates `{}` by adding fields      |
| `[{a: 1}], [{b: 2}]` | `[{a: 1, b: 2}]`   | Yes       | Attenuates existing caveat            |
| `[{a: 1}]`           | `[{a: 1}, {a: 2}]` | Yes       | Adds new caveat inside an `AND` block |
| `[{a: 1}]`           | `[{}]]`            | No        | Escalation by removing fields         |
| `[{a: 1}]`           | `[{b: 2}]`         | No        | Escalation by replacing fields        |

## 5.4 Signature Validation

The `sig` field MUST validate against the `iss` DID from the [Payload].

# 6. Acknowledgments

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

<!-- Footnotes -->

[^js-num-size]: JavaScript has a single numeric type ([`Number`][JS Number]) for both integers and floats. This representation is defined as a [IEEE-754] double-precision floating point number, which has a 53-bit significand.

<!-- Internal Links -->

[Ability]: #43-ability
[Caveat]: #45-caveats
[Envelope]: #2-delegation-envelope
[Meta]: #35-meta
[Payload]: #3-delegation-payload
[Subject]: #41-subject
[True Caveat]: #441-the-true-caveat
[Wildcard Ability]: #4312--aka-wildcard
 
<!-- External Links -->

[Alan Karp]: https://github.com/alanhkarp
[Benjamin Goering]: https://github.com/gobengo
[Blaine Cook]: https://github.com/blaine
[Bluesky]: https://blueskyweb.xyz/
[Brendan O'Brien]: https://github.com/b5
[Brian Ginsburg]: https://github.com/bgins
[Brooklyn Zelenka]: https://github.com/expede 
[CIDv1]: https://github.com/multiformats/cid?tab=readme-ov-file#cidv1
[Christine Lemmer-Webber]: https://github.com/cwebber
[Christopher Joel]: https://github.com/cdata
[DAG-CBOR]: https://ipld.io/specs/codecs/dag-cbor/spec/
[DID fragment]: https://www.w3.org/TR/did-core/#terminology
[DID]: https://www.w3.org/TR/did-core/
[Dan Finlay]: https://github.com/danfinlay
[Daniel Holmgren]: https://github.com/dholms
[ES256]: https://www.rfc-editor.org/rfc/rfc7518#section-3.4
[EdDSA]: https://en.wikipedia.org/wiki/EdDSA 
[Executor]: https://github.com/ucan-wg/spec#31-roles
[Fission]: https://fission.codes
[Hugo Dias]: https://github.com/hugomrdias
[IEEE-754]: https://ieeexplore.ieee.org/document/8766229
[IPLD]: https://ipld.io/
[Ink & Switch]: https://www.inkandswitch.com/
[Invocation]: https://github.com/ucan-wg/invocation
[Irakli Gozalishvili]: https://github.com/Gozala
[JS Number]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number
[Juan Caballero]: https://github.com/bumblefudge
[Mark Miller]: https://github.com/erights
[Martin Kleppmann]: https://martin.kleppmann.com/
[Mikael Rogers]: https://github.com/mikeal/
[OCAP]: https://en.wikipedia.org/wiki/Object-capability_model
[OCapN]: https://github.com/ocapn/
[Command]: https://github.com/ucan-wg/spec#33-command
[Philipp Krüger]: https://github.com/matheus23
[PoLA]: https://en.wikipedia.org/wiki/Principle_of_least_privilege 
[Protocol Labs]: https://protocol.ai/
[RFC 3339]: https://www.rfc-editor.org/rfc/rfc3339
[RFC 8037]: https://www.rfc-editor.org/rfc/rfc8037
[RS256]: https://www.rfc-editor.org/rfc/rfc7518#section-3.3
[Raw data multicodec]: https://github.com/multiformats/multicodec/blob/master/table.csv#L41
[SHA2-256]: https://github.com/multiformats/multicodec/blob/master/table.csv#L9
[SPKI/SDSI]: https://datatracker.ietf.org/wg/spki/about/
[SPKI]: https://theworld.com/~cme/html/spki.html
[Steven Vandevelde]: https://github.com/icidasset
[UCAN Invocation]: https://github.com/ucan-wg/invocation
[UCAN]: https://github.com/ucan-wg/spec
[W3C]: https://www.w3.org/
[ZCAP-LD]: https://w3c-ccg.github.io/zcap-spec/
[`did:key`]: https://w3c-ccg.github.io/did-method-key/
[`did:plc`]: https://github.com/did-method-plc/did-method-plc
[`did:web`]: https://w3c-ccg.github.io/did-method-web/
[base32]: https://github.com/multiformats/multibase/blob/master/multibase.csv#L13
[dag-json multicodec]: https://github.com/multiformats/multicodec/blob/master/table.csv#L112
[did:key ECDSA]: https://w3c-ccg.github.io/did-method-key/#p-256
[did:key EdDSA]: https://w3c-ccg.github.io/did-method-key/#ed25519-x25519
[did:key RSA]: https://w3c-ccg.github.io/did-method-key/#rsa
[external resource]: https://github.com/ucan-wg/spec#55-wrapping-existing-systems
[revocation]: https://github.com/ucan-wg/revocation
[ucan.xyz]: https://ucan.xyz
