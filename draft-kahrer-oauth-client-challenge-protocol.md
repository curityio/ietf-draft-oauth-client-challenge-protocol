---
title: "OAuth client Challenge Protocol"
category: std

docname: draft-kahrer-oauth-client-challenge-protocol-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: TBD
keyword:
 - oauth
 - client
 - client challenge
 - additional input

venue: # About this Document/Discussion Vanue (to be removed before publishing as an RFC)
  #group: "Web Authorization Protocol" # Name of Working or Research Group that this draft targets or that accepted the individual draft
  #type: "Working Group" # WG on IETF, RG on IRTF
  #mail: "oauth@ietf.org" # Mailing list for discussions
  #home: # Link to more documentation or project home
  #arch: "https://mailarchive.ietf.org/arch/browse/oauth/" # Archive for (mail) discussions
  github: "curityio/ietf-draft-oauth-client-challenge-protocol" # Source for this draft and issue tracker
  #latest: "https://curityio.github.io/ietf-draft-oauth-client-challenge-protocol/draft-kahrer-oauth-client-challenge-protocol.html" # Where to find the latest revision of this draft (github pages)

author:
 -
    fullname: Judith Kahrer
    organization: Curity
    email: judith.kahrer@curity.io

normative:

informative:

...

--- abstract

This document extends the OAuth 2.0 token endpoint error response (RFC 6749) with a new error code that indicates to the client that it must provide additional input for the authorization server to authorize it and accept its request.

This mechanism enables just-in-time authorization flows in which the authorization server dynamically challenges the client during a request, for example, to obtain an assertion, a Verifiable Presentation, or other proof-of-possession material mid-flow without an end-user being present.

--- middle

# Introduction

The OAuth 2.0 Authorization Framework {{!RFC6749}} assumes that whenever a Resource Owner needs to provide a grant, the client
can trigger an interactive flow to get that grant which delegates access to the client. Within those assumptions, the authorization server can utilize user prompts to increase the confidence in the grant because interactive flows imply that the Resource Owner is present. However, this is not always the case. clients may act on the behalf of a Resource Owner without the Resource Owner being present. In such cases, an interactive flow is not applicable to collect a grant or request additional input.

The OAuth 2.0 Authorization Framework {{!RFC6749}} also assumes that a single grant signaled through the `grant_type` parameter is sufficient for the authorization server to authorize the client. It does not define how the authorization server signals to the client to provide additional input for it to make a decision - like an additional grant from the Resource Owner or an attestation artifact to prove the client's provenance.

Real-world deployments increasingly require the authorization server to apply dynamic, contextual authorization policies — for example:

- Demanding a freshly signed client attestation when risk signals indicate an elevated threat level.
- Requiring the client to prove its mandate before a high-value token is issued.

This document extends the OAuth 2.0 Authorization Framework by introducing:

1. A new error code `insufficient_client_authorization` that the authorization server returns when it cannot proceed without additional client-supplied material.
2. A companion response parameter `authorization_requirement` — a typed JSON object
   that describes what the authorization server requires.
3. Processing rules for both parties, including the requirement to return
   `unauthorized_client` when subsequently provided input fails validation.

## Comparison with OAuth 2.0 First-Party Applications

OAuth 2.0 First-Party Applications {{?I-D.ietf-oauth-first-party-apps}} defines an API for user authentication where the authorization server challenges the OAuth 2.0 client to provide data from the user. The proposed API is similar to the mechanism defined in this document. However, there is a subtle difference: OAuth 2.0 First-Party Applications defines a new error code for the client to provide more data from the end-user (Resource Owner). Its main purpose is to enable clients to control the user experience. For that it makes two important assumptions:

- The client can interact with an end-user.
- The client is trusted to handle sensitive data like the end-user's credentials, i.e., the client is a first-party application.

The extension in this document is different because it assumes that the client can satisfy the challenge from the Authorization Requirement itself. It is applicable for both first- and third-party use cases where the authorization server challenges the client to provide more input about itself without involving an end-user. The client does not have to handle end-user credentials. What's more, it does not require an additional endpoint but extends the Token Response. In this way, the extension specifically targets non-interactive OAuth flows between the client and authorization server.

## Motivation and Use Cases

### Just-in-Time Authorization

Whether and in which form an authorization server may return an access token to a client depends not only on the presented grant and requested access but also on the capabilities of the client, e.g., whether it is a public client or a confidential client, whether it is a first-party client or a third-party client. Just-in-time authorization means that the authorization server can, dynamically and on demand, evaluate necessary data from different sources to accept or deny a token request from a client.

In a just-in-time flow, the authorization server defers its final authorization decision, challenges the client for supplemental proof material, and only then either grants or denies the request.

### Client Authentication Step-Up

An authorization server may accept client authentication for low-assurance tokens but require an attestation for tokens granting elevated privileges (see ({{?I-D.ietf-oauth-attestation-based-client-auth}})).
The `insufficient_client_authorization` mechanism allows the authorization server to escalate the authentication requirement without the client needing to speculatively include high-assurance credentials on every request.

### Continuos Authorization

The security state of a system can change at any time. Systems may communicate via signals about certain security-relevant events that they observed for other systems to adopt. In such an environment the authorization server may receive signals that constitute the need for additional input for authorization to e.g., mitigate attacks and prevent misuse.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Terminology

This document uses the terms "Access Token", "Authorization Code", "Authorization Request", "authorization server", "client", "client Authentication", "Protected Resource", "Resource Server", "Token Response", and "Token Endpoint" as defined by the OAuth 2.0 Authorization Framework {{RFC6749}}, unless otherwise specified by this document.

In addition, the document uses the following terms:

**Insufficient Client Authorization Response**:
: An error response for the OAuth 2.0 token endpoint that indicates to the client that the authorization server requires additional input for it to authorize the client.

**Authorization Requirement**:
: A typed JSON object returned by the authorization server in an Insufficient Client Authorization Response that specifies the additional material the client must supply.

**Challenge Session**:
: A string managed by the authorization server that serves as the nonce in the challenge-response pattern. It associates an Insufficient Client Authorization Response with the subsequent request that satisfies it.

# Insufficient Client Authorization Response {#error-response}

This document registers the error code `insufficient_client_authorization` for use in OAuth 2.0 token endpoint error responses as defined in Section 5.2 of {{RFC6749}}.

The following content applies to the Insufficient Client Authorization Response.

- `error`: REQUIRED. The `error` parameter MUST be `insufficient_client_authorization`.
- `authorization_requirement`: REQUIRED. The `authorization_requirement` parameter is a typed JSON object as defined in {{authorization-requirement}}.

The authorization server MUST comply with Section 5.2 of {{RFC6749}}. The authorization server SHOULD respond with HTTP status code `403 (Forbidden)`. It MAY include other parameters in the response. The client MUST ignore any parameters it does not understand.

If the client does not understand or cannot satisfy the Authorization Requirement, it MUST treat the Insufficient Client Authorization Response as if the authorization server returned an `unauthorized_client` error.

The Insufficient Client Authorization Response indicates the following:

1. The authorization server has determined that it cannot yet authorize the client or issue an access token.
2. The condition is resolvable: the authorization server knows what additional input would allow it to proceed.
3. The authorization server wishes to challenge the client to supply that input.

The following represents a non-normative example of an Insufficient Client Authorization Response.

~~~
HTTP/1.1 403 Forbidden
Content-Type: application/json
Cache-Control: no-store

{
  "error": "insufficient_client_authorization",
  "authorization_requirement": {
    "type": "verifiable_presentation",
    "challenge_session": "7f3d9e2a-4c1b-4f8e-b5a0-1e6c8d2f0a9b",
    "presentation_definition": { ... }
  }
}
~~~

# The `authorization_requirement` Object {#authorization-requirement}

The `authorization_requirement` parameter holds a JSON object that indicates what type of input the authorization server requires for the client to satisfy the insufficient client authorization.

The following members are defined for all `authorization_requirement` types:

**type** (string, REQUIRED):
: An absolute URI or a registered string identifying the authorization requirement type.
  The value determines the semantics of other members in the object.

**challenge_session** (string, REQUIRED):
: An opaque identifier generated by the authorization server that binds this authorization challenge to the follow-up request.
  The client MUST include this value in the subsequent request to the authorization server if it receives one along with the `insufficient_client_authorization` error response.

**expires_in** (integer, OPTIONAL):
: Number of seconds from the time of the Insufficient Client Authorization Response until the `challenge_session` expires.
  The client MUST NOT submit a response after this time.

# Providing Authorization Requirement

Each profile of this document that specifies a type of Authorization Requirement also MUST define how the client can fulfill the challenge and provide the required input to the authorization server.

If the client does not understand the `type` of the `authorization_requirement` of an Insufficient Client Authorization Response or if it cannot satisfy the requirement, the client MUST treat the Insufficient Client Authorization Response as if the authorization server returned an `unauthorized_client` error as defined in Section 5.2 in {{!RFC6749}}, see also {{error-response}}.

Some extensions to OAuth 2.0, notably Pushed Authorization Requests {{?RFC9126}}, make use of the token endpoint response outside a token endpoint request. A profile that defines an Authorization Requirement type SHOULD define mechanisms to fulfill the requirements that are applicable to authorization and token requests alike.

If the authorization server deems the supplied input from the client in response to an Authorization Requirement challenge as invalid and if there is no other way for the client to resolve the `insufficient_authorization` error, the authorization server MUST respond with an `unauthorized_client` error.

# Security Considerations

TODO Security


# IANA Considerations

TODO: Update IANA actions. Add registration for `insufficient_client_authorization`, `authorization_requirement`.

--- back

# Appendix
{:numbered="false"}

## Example with Authorization Details
{:numbered="false"}

The following is an example that illustrates a token request using `authorization_details` as defined in {{?RFC9396}}.
The authorization details indicate that the client aims to operate in an open banking ecosystem that has certain requirements.

~~~
POST /token HTTP/1.1
Host: authorization-server.example
Content-Type: application/x-www-form-urlencoded
OAuth-Client-Attestation: eyJ0eXAiOiJvYX...
DPoP: eyJ0eXAiOiJkcG9...

grant_type=client_credentials&
resource=https://api.openbanking.example&
authorization_details=%5B%7B%22type%22%3A%22payment_initiation%22%2C%22actions%22%3A%5B%22initiate%22%2C%22status%22%2C%22cancel%22%5D%2C%22locations%22%3A%5B%22https%3A%2F%2Fapi.openbanking.example%2Fpayments%22%5D%2C%22instructedAmount%22%3A%7B%22currency%22%3A%22EUR%22%2C%22amount%22%3A%22123.50%22%7D%2C%22creditorName%22%3A%22Merchant%20A%22%2C%22creditorAccount%22%3A%7B%22iban%22%3A%22DE02100100109307118603%22%7D%2C%22remittanceInformationUnstructured%22%3A%22Ref%20Number%20Merchant%22%7D%5D

~~~

The client needs to prove that it complies with the requirements, so the authorization server challenges the client and responds with a Insufficient Client Authorization Response. The authorization server requests from the client the presentation of a verifiable credentials, e.g., a verifiable intent ([Verifiable Intent (VI) — Specification Overview](https://verifiableintent.dev/spec/)).

~~~
HTTP/1.1 403 Forbidden
Content-Type: application/json
Cache-Control: no-store

{
  "error": "insufficient_client_authorization",
  "authorization_requirement": {
    "type": "verifiable_presentation",
    "challenge_session": "7f3d9e2a-4c1b-4f8e-b5a0-1e6c8d2f0a9b",
    "presentation_definition": { ... }
  }
}
~~~

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
