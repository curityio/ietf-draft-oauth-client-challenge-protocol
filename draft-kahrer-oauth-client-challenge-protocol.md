---
title: "OAuth Client Challenge Protocol"
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

This document extends the OAuth 2.0 token endpoint error response (RFC 6749) with a new error code that indicates to the client that it must provide additional input for the Authorization Server to authorize it and accept its request.

This mechanism enables just-in-time authorization flows in which the Authorization Server dynamically challenges the Client during a request, for example, to obtain an assertion, a Verifiable Presentation, or other proof-of-possession material mid-flow without an end-user being present.

--- middle

# Introduction

The OAuth 2.0 Authorization Framework {{!RFC6749}} assumes that whenever a Resource Owner needs to provide a grant, the Client
can trigger an interactive flow to get that grant which delegates access to the Client. Within those assumptions, the Authorization Server can utilize user prompts to increase the confidence in the grant because interactive flows imply that the Resource Owner is present. However, this is not always the case. Clients may act on the behalf of a Resource Owner without the Resource Owner being present. In such cases, an interactive flow is not applicable to collect a grant or request additional input.

The OAuth 2.0 Authorization Framework {{!RFC6749}} also assumes that a single grant signaled through the `grant_type` parameter is sufficient for the Authorization Server to authorize the Client. It does not define how the Authorization Server signals to the Client to provide additional input for it to make a decision - like an additional grant from the Resource Owner or an attestation artifact to prove the Client's provenance.

Real-world deployments increasingly require the Authorization Server to apply dynamic, contextual authorization policies — for example:

- Demanding a freshly signed client attestation when risk signals indicate an elevated threat level.
- Requiring the Client to prove its mandate before a high-value token is issued.

This document extends the OAuth 2.0 Authorization Framework by introducing:

1. A new error code `insufficient_client_authorization` that the Authorization Server returns when it cannot proceed without additional client-supplied material.
2. A companion response parameter `authorization_requirement` — a typed JSON object
   that describes what the Authorization Server requires.
3. Processing rules for both parties, including the requirement to return
   `unauthorized_client` when subsequently provided input fails validation.

## Comparison with OAuth 2.0 First-Party Applications

OAuth 2.0 First-Party Applications {{?I-D.ietf-oauth-first-party-apps}} defines an API for user authentication where the Authorization Server challenges the OAuth 2.0 Client to provide data from the user. The proposed API is similar to the mechanism defined in this document. However, there is a subtle difference: OAuth 2.0 First-Party Applications defines a new error code for the Client to provide more data from the end-user (Resource Owner). Its main purpose is to enable Clients to control the user experience. For that it makes two important assumptions:

- The Client can interact with an end-user.
- The Client is trusted to handle sensitive data like the end-user's credentials, i.e., the Client is a first-party application.

The extension in this document is different because it assumes that the Client can satisfy the challenge from the Authorization Requirement itself. It is applicable for both first- and third-party use cases where the Authorization Server challenges the Client to provide more input about itself without involving an end-user. The Client does not have to handle end-user credentials. What's more, it does not require an additional endpoint but extends the Token Response. In this way, the extension specifically targets non-interactive OAuth flows between the Client and Authorization Server.

## Motivation and Use Cases

### Just-in-Time Authorization

Traditional OAuth 2.0 flows resolve all authorization decisions before the Client calls the Token Endpoint.
However, certain policy frameworks — notably those aligned with Zero Trust Architecture or dynamic risk-based access control — require the Authorization Server to evaluate context that becomes available only at the moment of the token request. This document calls that pattern *just-in-time authorization*.

In a just-in-time flow, the Authorization Server defers its final authorization decision, challenges the Client for supplemental proof material, and only then either grants or denies the request.

### Client Authentication Step-Up

An Authorization Server may accept client authentication for low-assurance token types but require an attestation for tokens granting elevated privileges (see ({{?I-D.ietf-oauth-attestation-based-client-auth}})).
The `insufficient_client_authorization` mechanism allows the Authorization Server to escalate the authentication requirement without the Client needing to speculatively include high-assurance credentials on every request.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Terminology

This document uses the terms "Access Token", "Authorization Code", "Authorization Request", "Authorization Server", "Client", "Client Authentication", "Protected Resource", "Resource Server", "Token Response", and "Token Endpoint" as defined by the OAuth 2.0 Authorization Framework {{RFC6749}}, unless otherwise specified by this document.

In addition, the document uses the following terms:

**Insufficient Client Authorization Response**:
: An error response for the OAuth 2.0 token endpoint that indicates to the Client that the Authorization Server requires additional input for it to authorize the Client.

**Authorization Requirement**:
: A typed JSON object returned by the Authorization Server in an Insufficient Client Authorization Response that specifies the additional material the Client must supply.

**Challenge Session**:
: A string managed by the Authorization Server that serves as the nonce in the challenge-response pattern. It associates an Insufficient Client Authorization Response with the subsequent request that satisfies it.

# Insufficient Client Authorization Response {#error-response}

This document registers the error code `insufficient_client_authorization` for use in OAuth 2.0 token endpoint error responses as defined in Section 5.2 of {{RFC6749}}.

The following content applies to the Insufficient Client Authorization Response.

- `error`: REQUIRED. The `error` parameter MUST be `insufficient_client_authorization`.
- `authorization_requirement`: REQUIRED. The `authorization_requirement` parameter is a typed JSON object as defined in {{authorization-requirement}}.

The Authorization Server MUST comply with Section 5.2 of {{RFC6749}}. This implies that the Authorization Server MUST respond with HTTP status code `400 (Bad Request)`. It MAY include other parameters in the response. The Client MUST ignore any parameters it does not understand.

If the Client does not understand or cannot satisfy the Authorization Requirement, it MUST treat the Insufficient Client Authorization Response as if the Authorization Server returned an `unauthorized_client` error.

The Insufficient Client Authorization Response indicates the following:

1. The Authorization Server has determined that it cannot yet authorize the Client or issue an access token.
2. The condition is resolvable: the Authorization Server knows what additional input would allow it to proceed.
3. The Authorization Server wishes to challenge the client to supply that input.

The following represents a non-normative example of an Insufficient Client Authorization Response.

~~~
HTTP/1.1 400 Bad Request
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

The `authorization_requirement` parameter holds a JSON object that indicates what type of input the Authorization Server requires for the Client to satisfy the insufficient client authorization.

The following members are defined for all `authorization_requirement` types:

**type** (string, REQUIRED):
: An absolute URI or a registered string identifying the authorization requirement type.
  The value determines the semantics of other members in the object.

**challenge_session** (string, REQUIRED):
: An opaque identifier generated by the Authorization Server that binds this authorization challenge to the follow-up request.
  The Client MUST include this value in the subsequent request to the Authorization Server if it receives one along with the `insufficient_client_authorization` error response.

**expires_in** (integer, OPTIONAL):
: Number of seconds from the time of the Insufficient Client Authorization Response until the `challenge_session` expires.
  The client MUST NOT submit a response after this time.

# Providing Authorization Requirement

Each profile of this document that specifies a type of Authorization Requirement also MUST define how the Client can fulfill the challenge and provide the required input to the Authorization Server.

If the Client does not understand the `type` of the `authorization_requirement` of an Insufficient Client Authorization Response or if it cannot satisfy the requirements, the Client MUST treat the Insufficient Client Authorization Response as if the Authorization Server returned an `unauthorized_client` error as defined in Section 5.2 in {{!RFC6749}}, see also {{error-response}}.

Some extensions to OAuth 2.0, notably Pushed Authorization Requests {{?RFC9126}}, make use of the token endpoint response outside a token endpoint request. A profile that defines an Authorization Requirement type SHOULD define mechanisms to fulfill the requirements that are applicable to authorization and token requests alike.

If the Authorization Server deems the supplied input from the Client in response to an Authorization Requirement challenge as invalid and if there is no other way for the Client to resolve the `insufficient_authorization` error, the Authorization Server MUST respond with an `unauthorized_client` error.

# Security Considerations

TODO Security


# IANA Considerations

TODO: Update IANA actions. Add registration for `insufficient_client_authorization`, `authorization_requirement`.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
