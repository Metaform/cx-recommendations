# EDC Implementation Tasks for the Identity And Trust Protocols

The specification documents can be found [here](https://github.com/eclipse-tractusx/identity-trust/).

This document outlines all necessary implementation tasks for EDC and associated components.

## For The Presentation Flow

### 1. EDC: Determine necessary/required scopes

The consumer connector needs to extract the required credentials for a particular request (contract negotiation,...),
convert it into
an [access scope](https://github.com/eclipse-tractusx/identity-trust/blob/main/specifications/M1/verifiable.presentation.protocol.md#31-access-scopes),
store it on the `TokenParameters#scope` field and feed that to the `IdentityService`. This mapping from the policy
constraint onto a scope must be _pluggable/extensible_

#### 1.a Subtask Tractus-X EDC: Map CredentialType to scope

In Catena-X, the policy constraint expressions contain
the [credential type](https://github.com/eclipse-tractusx/ssi-docu/blob/main/docs/architecture/cx-3-2/3.%20Verifiable%20Credentials/CX-Credentials/Standardized%20CX-Credential.md),
e.g. `DismantlerCredential`. Therefore, in Tractus-X EDC we need to implement an extension that maps the credential
type onto an access scope and vice versa.

#### 1.b Subtask EDC: use OAuth2 to obtain Self-Issued ID Token from STS

An identity service needs to be implemented (`IdentityAndTrustService` (ITS)), that requests the token from the STS (
using basic OAuth flows), but on the verification side it follows the presentation protocol.

### 2. EDC: Implement a Secure Token Service

EDC is to provide a self-contained implementation of the Secure Token Service, that
generates [self-issued ID tokens](https://github.com/eclipse-tractusx/identity-trust/blob/main/specifications/M1/identity.protocol.base.md#4-self-issued-id-tokens).

#### 2.a Subtask EDC: specify REST API for the STS

This is the API spec, that is basically OAuth2, that every STS must expose to create tokens.

#### 2.b Subtask EDC: implement self-contained SecureTokenService

EDC will implement a standalone runtime exposing the STS API (OAuth2), that can create tokens as per Identity And
Trust Protocols spec.
Requirements for the EDC implementation of the STS:

- create a self-issued ID JWT token, that contains an `access_token` within
- the `access_token` contains the aforementioned scopes as claims
- `sub` claim contains the provider's DID

#### 2.c Subtask EDC: implement short-circuit, when STS is embedded in EDC

The `IdentityAndTrustService` would by default call out to a (remote) STS. However, if the `edc.oauth.token.url`
config parameter isn't present, it can short-circuit the remote invocation and call out to an embedded STS as
fallback.

#### 2.d Subtask Tractus-X: adapt Helm charts to set the `edc.oauth.token.url` to MIW's Secure Token Service

### 3. Connector: Implement mechanism to guard against token leakage

When the provider connector sends back the consumer's ID token, it needs to take measure against a potential
leakage of that token such that the receiver (=consumer) can establish the provenance of the token. To that end, the
provider wraps the token in another token ("token-in-token"), signing it with its own private key.

#### 3.1 Subtask EDC: validate incoming ID Token from consumer

Obtain the consumer's public key from their DID document. The validation should follow basic OAuth2 rules. Reject the
token if validation fails. _This is already implemented._

#### 3.2 Subtask EDC: create TiT

The `IdentityAndTrustService` must generate the TiT (using STS) with the following parameters:

- `aud` claim contains the consumer's DID
- `iss` claim contains the provider's DID
- `sub` == `iss`
- `access_token` claim: the access_token that was encoded in the original ID token

#### 3.3 Subtask EDC: add code to send credentials request

The `IdentityAndTrustService` must resolve the consumer's DID Document to obtain its CS endpoint. For that, we can
re-use the existing `DidResolver`/`DidResolverRegistry`. It can then build an HTTP request, that contains the
credential query (using scopes, "prover scope") as body, contains the TiT as `Authentication` header and sends it to
the consumer's CS endpoint.

#### 3.4 Subtask EDC: verify TiT

Same as [3.1](#31-subtask-edc-validate-incoming-id-token-from-consumer), but on provider side. Note that the code to
validate the TiT would actually run in the [IdentityHub](#42-subtask-identityhub-implement-titverifier-working-title).

### 4. IdentityHub: Implement VP Query

Any CredentialService must expose
a [Resolution API](https://github.com/eclipse-tractusx/identity-trust/blob/main/specifications/M1/verifiable.presentation.protocol.md#411-query-for-presentations).

#### 4.1 Subtask IdentityHub: Create `PresentationApi|Controller`

Implement basic glue code to expose the Resolution API. That includes:

- An interface to carry OpenAPI annotations
- A controller class (delegates to the [verifier](#43-subtask-identityhub-implement-titverifier-working-title) and
  the [query resolver](#42-subtask-identityhub-implement-presentationqueryresolver))
- Model class `PresentationQuery` + `JsonObjectToPresentationQueryTransformer`
- Model class `VerifiablePresentation` + `JsonObjectFromVerifiablePresentationTransformer`

#### 4.2 Subtask IdentityHub: Implement `TitVerifier` (working title)

Takes the incoming Token-in-token and performs a series of verifications on it. Resolving the provider's DID document
and performing basic OAuth Token validation is equal to [3.1](#31-subtask-edc-validate-incoming-id-token-from-consumer),
but in addition, it performs these steps:

- `iss` == `sub`
- `aud` == own DID
- pull out access token, verify with own STS public key
- `access_token.sub` == `sub`: verify that the access token's `sub` claim equals the TiT's `sub` claim
- assert that scope can be obtained from access_token

The `TitVerifier` then returns the extracted scope, or an error result.

#### 4.3 Subtask IdentityHub: Implement `PresentationQueryResolver`

Defines a model class `VerifiableCredential`.

Takes "prover" scope (from the request body) and the "issues scope" (extracted from the TiT) and performs the
following steps:

- validate scopes: the "prover" scope may not be wider than the "issuer" scope
- transform query: converts the "prover" scope into Credential types (using the inverse
  of [1.a](#1a-subtask-tractus-x-edc-map-credentialtype-to-scope)). Note that the scope has the credential format
  encoded. A query that specifies VCs in different formats is invalid (homogeneity check).
  See [section 5.1](#51-a-note-on-vcvp-formats) for details.
- compose a `QuerySpec` based on those credential type(s).
- execute query. Convert DB entities into a `List<VerifiableCredential>`

### 5. Connector: Implement VP Generator module

Generates a verifiable presentation out of a list of verifiable credentials:

```java
public interface PresentationGenerator {

    /**
     * Generates a VP out of one or several VCs
     * @param credentials The list of VerifiableCredentials
     * @param cryptoSuite A Crypto Suite used to create the proof, Jws2020 or Ed25519
     * @param format The credential format, JWT or JSON_LD
     */
    Result<VerifiablePresentation> createPresentation(List<VerifiableCredential> credentials, CryptoSuite cryptoSuite,
                                                      CredentialFormat format);
}
```

#### 5.1 A note on VC/VP formats

Verifiable credentials and presentations are available in two formats, JSON-LD with linked proofs and as JWT. While it
is theoretically possible to mix-and-match, the EDC IdentityHub will always generate "homogenous" VPs, i.e. all the VCs
must be represented in the same format, and the `format` parameter must match that format.

Due to the homogeneity check performed in [query resolver](#43-subtask-identityhub-implement-presentationqueryresolver)
it can be
guaranteed that all credentials are of the same format.

However, if the `format` parameter is different from all the VCs' formats, the method would return a failure result.

> Note 1: Initially, the `cryptoSuite` and `format` parameter will be ignored, and `Jws2020` and `JSON_LD` will be used
> respectively.

### 6. Connector: Implement VP Validation module

In EDC we already have a `CredentialsVerifier`, which takes a `DidDocument`. This may not be perfectly suitable here,
because the input parameter type should be `VerifiablePresentation`.

In EDC we will implement a class that:
- performs basic VP validation (proofs) etc. For that, the `CryptoSuite` should be used.
- for every credential in the VP:
  - validates `subjectId` of every VC: must match the consumer DID ("did the consumer only send their own VCs?")
  - check credential against the revocation list
  - check that issuer is in the list of [allowed issuers](#1-supporting-multiple-trusted-issuers)
  - validates credential properties, i.e. that it contains the required information


## The Issuance Flow

TBD

## General implementation tasks
### 1. Supporting multiple trusted issuers