# UCAN Delegation Specification v1.0.0-rc.1

## Editors

- [Brooklyn Zelenka], [Witchcraft Software]

## Authors

- [Brooklyn Zelenka], [Witchcraft Software]
- [Daniel Holmgren], [Bluesky]
- [Irakli Gozalishvili], [Protocol Labs]
- [Philipp Krüger], [number zero]

## Dependencies

- [UCAN]

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119].

# Abstract

This specification describes the representation and semantics for delegating attenuated authority between principals. UCAN Delegation provides a cryptographically verifiable container, batched capabilities, hierarchical authority, and a minimal syntatically-driven policy langauge.

# Introduction

UCAN Delegation is a delegable certificate capability system with runtime-extensibility, ad hoc conditions, cacheability, and focused on ease of use and interoperability. Delegations act as a proofs for [UCAN Invocation]s.

Delegation provides a way to "transfer authority without transferring cryptographic keys". As an authorization system, it is more interested in "what can be done" than a list of "who can do what". For more on how Delegation fits into UCAN, please refer to the [high level spec][UCAN].

# [UCAN Envelope] Configuration

## Type Tag

The UCAN envelope tag for UCAN Delegation MUST be set to `ucan/dlg@1.0.0-rc.1`.
 
## Delegation Payload

The Delegation payload MUST describe the authorization claims, who is involved, and its validity period.

| Field   | Type                                     | Required | Description                                                 |
|---------|------------------------------------------|----------|-------------------------------------------------------------|
| `iss`   | `DID`                                    | Yes      | Issuer DID (sender)                                         |
| `aud`   | `DID`                                    | Yes      | Audience DID (receiver)                                     |
| `sub`   | `DID`                                    | Yes      | Principal that the chain is about (the [Subject])           |
| `cmd`   | `String`                                 | Yes      | The [Command] to eventually invoke                          |
| `pol`   | `Policy`                                 | Yes      | [Policy]                                                    |
| `nonce` | `Bytes`                                  | Yes      | Nonce                                                       |
| `meta`  | `{String : Any}`                         | No       | [Meta] (asserted, signed data) — is not delegated authority |
| `nbf`   | `Integer` (53-bits[^js-num-size])        | No       | "Not before" UTC Unix Timestamp in seconds (valid from)     |
| `exp`   | `Integer \| nul` (53-bits[^js-num-size]) | Yes      | Expiration UTC Unix Timestamp in seconds (valid until)      |

# Capability

Capabilities are the semantically-relevant claims of a delegation. They MUST be presented as a map under the `cap` field as a map. This map is REQUIRED but MAY be empty. This MUST take the following form:

| Field  | Type      | Required | Description                                                                                      |
|--------|-----------|----------|--------------------------------------------------------------------------------------------------|
| `sub`  | `DID`     | Yes      | The [Subject] that this Capability is about                                                      |
| `cmd`  | `Command` | Yes      | The [Command] of this Capability                                                                 |
| `pol`  | `Policy`  | Yes      | Additional constraints on eventual Invocation arguments, expressed in the [UCAN Policy Language] |

Here is an illustrative example:

``` js
{
  // ...
  "sub": "did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp"
  "cmd": "/blog/post/create",
  "pol": [
    ["==", ".status", "draft"],
    ["every", ".reviewer", ["match", ".email", "*@example.com"]],
    ["some", ".tags", 
      ["or",
        ["==", ".", "news"], 
        ["==", ".", "press"]]]
  ]
}
```

## Subject

The Subject MUST be the DID that initiated the delegation chain.

For example:

``` js
{
  "sub": "did:web:example.com",
  // ...
}
```

## Resource

Unlike Subjects and Commands, Resources are semantic rather than syntactic. The Resource is the "what" that a capability describes.

By default, the Resource of a capability is the Subject. This makes the delegation chain self-certifying.

``` js
{
  "sub": "did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp", // Subject & Resource
  "cmd": "/crud/update",
  // ...
}
```

In the case where access to an [external resource] is delegated, the Subject MUST own the relationship to the Resource. The Resource SHOULD be referenced by a `uri` key in the relevant [Conditions], except where it would be clearer to do otherwise. This MUST be defined by the Subject and understood by the executor.

``` js
{
  "sub": "did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp",
  "cmd": "/crud/create",
  "pol": [
    ["==", ".uri", "https://example.com/blog/"], // Resource
    ["==", ".status", "draft"]
  ],
  // ...
}
```

# Policy Language

UCAN Delegation uses a minimal predicate language as a policy language. Policies are syntactically driven, and MUST constrain the `args` field of an eventual [Invocation].


The grammar of the UCAN Policy Language is as follows:

## Syntax


``` abnf
policy      = "[" *predicate "]"
predicate   = equality 
            / inequality 
            / match
            / connective 
            / quantifier

;; STRUCTURAL

connective  = "['not', " policy "]" ; Negation
            / "['and', " policy "]" ; Conjuction
            / "['or',  " policy "]" ; Disjunction

quanitifier = "['every', " selector ", " policy "]" ; Universal
            / "['some', "  selector ", " policy "]" ; Existential

;; COMPARISONS

equality    = "['==', " selector ", " ipld   "]" ; Equality on IPLD literals
inequality  = "['>', "  selector ", " number "]" ; Numeric greater-than
            / "['>=', " selector ", " number "]" ; Numeric greater-than-or-equal
            / "['<',  " selector ", " number "]" ; Numeric lesser-than
            / "['<=', " selector ", " number "]" ; Numeric lesser-than-or-equal

match       = "['match', " selector ", " pattern "]" ; String wildcard matching

;; SELECTORS

selector    = "."                    ; Identity
            / *(".") 1*(subselector) ; Nested subselectors

subselector = "." 1*utf8       ; Dotted field selector
            / "['" string "']" ; Explicit field selector
            / "["  integer "]" ; Index selector

;; SPECIAL LITERALS

pattern     = *utf8
number      = integer / float
```

### Pattern 

## Semantics

A Policy is always given as an array of predicates. This top-level array is implicitly treated as a logical `and`, where `args` MUST pass validation of every top-level predicate.

Policies are structured as trees. With the exception of subtrees under `some`, `or`, and `not`, every leaf MUST evaluate to `true`. `some`, `or`, and `not` must 

### Selectors

Selector syntax is closely based on a subset of [jq]. They operate on an [Invocation]'s `args` object.

For example, consider the following:

``` js
// Invocation
{
  "args": {
    "from": "alice@example.com",
    "to": ["bob@example.com", "carol@elsewhere.example.com"],
    "cc": ["fraud@example.com"],
    "title": "Meeting Confirmation",
    "body": "I'll see you on Tuesday"
  },
  // ...
}
```

<table>
  <thead>
    <tr>
      <td>Selector</td>
      <td>Value</td>
    </tr>
  </thead>
  
  <tr>
    <td>`.`</td>
    <td>
      ```js
        {
          "from": "alice@example.com",
          "to": ["bob@example.com", "carol@elsewhere.example.com"],
          "cc": ["fraud@example.com"],
          "title": "Meeting Confirmation",
          "body": "I'll see you on Tuesday"
        }
      ```
    </td>
  </tr>

  <tr>
    <td>`.title`</td>
    <td>`"Meeting Confirmation"`</td>
  </tr>
  
  <tr>
    <td>`.to[1]`</td>
    <td>`"carol@elsewhere.example.com"`</td>
  </tr>
</table>

### Validation

## Examples

``` js
// Delegation
{
  "cmd": "/msg/send",
  "pol": [
    ["==", ".from", "alice@example.com"
    ["==", ".title", "Coffee"]
  ],
  // ...
}

// Valid invocation
{
  "cmd": "/msg/send",
  "args": {
    "from": "alice@example.com", // Matches above
    "to": ["bob@example.com", "carol@elsewhere.example.com"],
    "title": "Coffee",
    "body": "Still on for coffee" // Matches above
  },
  // ...
}
```

Any arguments MUST be taken verbatim and MUST NOT be further adjusted. For more flexible validation of Arguments, use [Conditions].

Note that this also applies to arrays and objects. For example, the `to` array in this example is considered to be exact, so the Invocation fails validation in this case:

``` js
// Delegation
{
  "cmd": "/msg/send",
  "pol": [
    ["==", ".from", "alice@example.com"],
    ["some", ".to", ["match", ".", "*@example.com"]]
  ]
  // ...
}

// VALID Invocation
{
  "cmd": "/msg/send",
  "args": {
    "from": "alice@example.com",
    "to": ["bob@example.com", "carol@elsewhere.example.com"],
    "title": "Coffee",
    "body": "Still on for coffee"
  },
  // ...
}

// INVALID Invocation
{
  "cmd": "/msg/send",
  "args": {
    "from": "alice@example.com",
    "to": ["carol@elsewhere.example.com"], // Missing `*@example.com`
    "title": "Coffee",
    "body": "Still on for coffee"
  },
  // ...
}
```

The intended logic is expressible with [Conditions].

## Conditions

// FIXME
The `cond` field MUST contain any additional conditions. This concept is sometimes called a "caveat". Conditions constrain the capability in two ways:

- Syntactic constraints on [Arguments] (length, regex, inclusion)
- Environmental / contextual conditions (day of week)

Condition semantics MUST be established by the Subject. They are openly extensible, but vocabularies may be reused across many Subjects. Conditions MUST be Understood by the [Executor] of the eventual [Invocation][UCAN Invocation]. Each Condition MUST be formatted as a map.

``` js
// Delegation
{
  "sub": "did:web:example.com",
  "cmd": "msg/send",
  "args": {
    "from": "alice@example.com"
  }
  "pol": ["or",
    // no recipient can have a `@gmail.com` email address
    ["every", ".to", ["not", ["match", ".", "*@competitor.example.com"]]],

    // or at least one `bcc` field must include @ourteam.example.com
    ["some", ".bcc", ["match", ".", "*@ourteam.example.com"]]
  ],
  // ...
}

// Valid Invocation, if Monday
{
  "do": "msg/send",
  "args": {
    "from": "alice@example.com",
    "to": ["bob@example.com", "carol@elsewhere.example.com"], // Matches criteria
    "title": "Coffee",
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
    a: 1, // ├─ Confition ─┐
    b: 2  // │             │
  },      // ┘             ├─ AND ─┐
  {       // ┐             │       │
    c: 3  // ├─ Condition ─┘       │
  },      // ┘                     ├─ AND
  {       // ┐                     │
    d: 4  // ├─ Condition ─────────┘
  }       // ┘
]
```

Expressing Conditions in this standard way simplifies ad hoc extension at delegation time. As a concrete example, below a Condition is added to the constraints on the `to` field.

``` js
// Original Condition
[
  {
    "field": "to",
    "includes": "bob@example.com"
  }
]

// Attenuated Conditions
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

# Validation

Validation of a UCAN chain MAY occur at any time, but MUST occur upon receipt of an [Invocation] prior to execution. While proof chains exist outside of a particular delegation (and are made concrete in [UCAN Invocation]s), each delegate MUST store one or more valid delegations chains for a particular claim.

Each capability has its own semantics, which needs to be interpretable by the [Executor]. Therefore, a validator MUST NOT reject all capabilities when one that is not relevant to them is not understood. For example, if a Condition fails a delegation check at execution time, but is not relevant to the invocation, it MUST be ignored.

If _any_ of the following criteria are not met, the UCAN MUST be considered invalid:

## Time Bounds

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

## Principal Alignment

In delegation, the `aud` field of every proof MUST match the `iss` field of the UCAN being delegated to. This alignment MUST form a chain back to the Subject for each resource. 

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

### Recipient Validation

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

## Condition Attenuation

FIXME

The Condition array MAY be empty (which is equivalent to saying "with no other conditions"). Delegations MUST otherwise only append more Conditions, and recapitulate the existing ones verbatim. Here are some abstract examples:

FIXME

Conditions MAY be presented in any order, but merely appending to the array is RECOMMENDED.

## Signature Validation

The [Signature] field MUST validate against the `iss` DID from the [Payload].

# Acknowledgments

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

[Ability]: #ability
[Envelope]: #delegation-envelope
[Meta]: #meta
[Payload]: #delegation-payload
[Subject]: #subject
[Policy]: #policy-language
[Policy Language]: #policy-language
 
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
[Command]: https://github.com/ucan-wg/spec#33-command
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
[Philipp Krüger]: https://github.com/matheus23
[PoLA]: https://en.wikipedia.org/wiki/Principle_of_least_privilege 
[Protocol Labs]: https://protocol.ai/
[RFC 3339]: https://www.rfc-editor.org/rfc/rfc3339
[RFC 8037]: https://www.rfc-editor.org/rfc/rfc8037
[RS256]: https://www.rfc-editor.org/rfc/rfc7518#section-3.3
[Raw data multicodec]: https://github.com/multiformats/multicodec/blob/master/table.csv#L41
[Revocation]: https://github.com/ucan-wg/revocation
[SHA2-256]: https://github.com/multiformats/multicodec/blob/master/table.csv#L9
[SPKI/SDSI]: https://datatracker.ietf.org/wg/spki/about/
[SPKI]: https://theworld.com/~cme/html/spki.html
[Steven Vandevelde]: https://github.com/icidasset
[UCAN Envelope]: https://github.com/ucan-wg/spec/blob/high-level/README.md#envelope
[UCAN Invocation]: https://github.com/ucan-wg/invocation
[UCAN]: https://github.com/ucan-wg/spec
[Varsig]: https://github.com/ChainAgnostic/varsig
[W3C]: https://www.w3.org/
[Witchcraft Software]: https://github.com/expede
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
[jq]: https://jqlang.github.io/jq/
[number zero]: https://n0.computer/
[revocation]: https://github.com/ucan-wg/revocation
[ucan.xyz]: https://ucan.xyz
