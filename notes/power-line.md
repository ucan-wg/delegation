# Delegation

## Root Cert

```js
[
  {"/": {"bytes": {"0xSignature"}}, // You must check this, and they're really small (except RSA)
  [
    {"/": {"bytes": {"0xVarsigHeader"}}, // Array to force this to the front
    {
      "ucan/1.0/d": { // "This is a v1.0 UCAN delegation"
        "sub": "did:key:alice",
        "iss": "did:key:alice",
        "aud": "did:key:bob",
        "cmd": "/http",
        "cond": [
          {"match": {"url": "https://example.com/api/v1/*"}},
          {
            "or": [
              {"==": {"headers": {"accept": "application/json"}}}}, // `==` is a kind of match 🤔
              {"==": {"headers": {"content-type": "application/json"}}
             ]
           }
        ],
        "exp": 1234567890,
        "meta": {
          "tags": ["development", "test-suite", "protocol-labs"]
        }
      }
    }
  ]
]
```

## Inner Chain — Normal

```js
[
  {"/": {"bytes": {"0xSignature"}},
  [
    {"/": {"bytes": {"0xVarsigHeader"}},
    {
      "ucan/1.0/d": {
        "sub": "did:key:alice",
        "iss": "did:key:bob",
        "aud": "did:key:carol",
        "cmd": "/http/get", // Attenuated
        "cond": [
          {"match": {"url": "https://example.com/api/v1/users/*"}} // Attenuated
          // "SHOULD" include the headers here too as a best practice, but no longer required in the spec
        ],
        "exp": 1234567890,
        "meta": {
          "tags": ["development", "test-suite", "protocol-labs"]
        }
      }
    }
  ]
]
```

## Inner Chain — Powerline

Because it confuses some people with traditional capabilties backgrounds, I'm going to _try_ calling our modified powerbox-like pattern a "powerline" because it works on delegation chains.

```js
[
  {"/": {"bytes": {"0xSignature"}},
  [
    {"/": {"bytes": {"0xVarsigHeader"}},
    {
      "ucan/1.0/d": {
        // No sub field
        "iss": "did:key:carol",
        "aud": "did:key:dora",
        "cmd": "/http", // Any HTTP command
        "cond": [], // No conditions for this specific example
        "exp": 1234567890,
        "meta": {
          "tags": ["development", "test-suite", "protocol-labs"]
        }
      }
    }
  ]
]
```

Just another example not in the invocation chain example to show another case ⬇️

```js
[
  {"/": {"bytes": {"0xSignature"}},
  [
    {"/": {"bytes": {"0xVarsigHeader"}},
    {
      "ucan/1.0/d": {
        // No aud field
        "iss": "did:key:paul",
        "aud": "did:key:sara",
        "cmd": "/dns", // Any DNS command
        "cond": [
          {"==": {"rr-type": "TXT"}} // Only TXT records
        ],
        "exp": 1234567890,
        "meta": {
          "tags": ["development", "test-suite", "protocol-labs"]
        }
      }
    }
  ]
]
```

# Invocation

```js
[
  {"/": {"bytes": {"0xSignature"}},
  [
    {"/": {"bytes": {"0xVarsigHeader"}},
    {
      "ucan/1.0/i": { // "This is a v1.0 UCAN invocation"
        "sub": "did:key:alice",
        "iss": "did:key:dora",
        "aud": "did:key:frank", // Optional field. If omitted, assumed to be the same as `aud`.
        "cmd": "/http/get",
        "args": {
          "url": "https://example.com/api/v1/users/42",
          "accept": "application/json"
        },
        "prf": [
          {"/": "bafy...carol-dora-powerline"},
          {"/": "bafy...bob-carol"},
          {"/": "bafy...alice-bob"}
        ],
        "exp": 1234567890,
        "meta": {
          "tags": ["development", "test-suite", "protocol-labs"]
        }
      }
    }
  ]
]
```
