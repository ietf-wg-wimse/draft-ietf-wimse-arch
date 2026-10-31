---
title: "Workload Identity in a Multi System Environment (WIMSE) Architecture"
abbrev: "WIMSE Architecture"
docname: draft-ietf-wimse-arch-latest
category: info
submissionType: IETF

ipr: trust200902
area: "Applications and Real-Time"
workgroup: "Workload Identity in Multi System Environments"
keyword: Internet-Draft

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: J. Salowey
    name: Joseph Salowey
    organization: Palo Alto Networks
    email: joe@salowey.net
 -
    ins: Y. Rosomakho
    name: Yaroslav Rosomakho
    organization: Zscaler
    email: yaroslavros@gmail.com
 -
    ins: H. Tschofenig
    name: Hannes Tschofenig
    organization: University of the Bundeswehr Munich
    abbrev: UniBw M.
    city: Neubiberg
    region: Bavaria
    country: Germany
    code: 85577
    email: hannes.tschofenig@gmx.net

normative:
informative:
   SPIFFE:
     title: Secure Production Identity Framework for Everyone (SPIFFE)
     target: https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE.md
     date: 10 May 2023
   SPIFFE-ID:
     title: The SPIFFE Identity and Verifiable Identity Document
     target: https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE-ID.md
     date: 21 May 2025
   SPIRE:
     title: SPIRE Concepts
     target: https://spiffe.io/docs/latest/spire-about/spire-concepts/

--- abstract

Applications increasingly consist of workloads running across different platforms and administrative domains. This document describes an architecture for workload identity and security context propagation to support authentication, authorization, and auditing.

--- middle

# Introduction

Cloud computing and microservices distribute application functions across workloads that need to communicate securely.

Workloads are provisioned with identity credentials when they start and may also receive trust anchors and security context information. They use these credentials to authenticate communications and use identity and context information to make authorization decisions.

This architecture considers two credential formats: X.509 certificates, commonly used with Transport Layer Security (TLS), and JSON Web Tokens (JWTs), used at the application layer. The choice depends on the application and security context.

When invoked, a workload may need to interact with other workloads. {{?I-D.ietf-oauth-transaction-tokens}} illustrates an externally facing endpoint invoked using a conventional authorization mechanism, such as an OAuth 2.0 access token. Security context associated with that authorization may need to be passed along the call chain.

This document defines the terminology, describes the architecture and its use cases, and discusses security considerations.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the following terms:

* Workload

A workload is an independently addressable and executable software entity that typically interacts with other parts of a larger system. It is a logical entity and need not correspond to a single running process. It may be implemented by one or more workload instances and may expose, consume, or participate in one or more services. Examples include microservices, containers, virtual machines (VMs), or serverless functions.

* Workload Instance

A workload instance is a single running instantiation of a workload, such as a container, a VM, or a serverless invocation. Its lifetime can range from a fraction of a second to handle a single request to months or years for database services or machine learning training jobs. The number of instances of a workload may vary over time due to scaling, failover, or orchestration.

* Service

A service is a function, API, or capability exposed to other entities. A service may be implemented by a single workload, by multiple workload instances of the same workload, or by multiple workloads acting together. A service can have a stable identity that is distinct from the identity of any particular workload instance. When this document refers to service authentication or service authorization, it refers to authentication or authorization of the workload identity used to represent that service in a particular deployment.

* Application System

An application system comprises workloads and supporting components, such as gateways, issuers, and token services, that implement an application or application function. It may span one or more trust domains.

* Security Context

A security context provides information needed for a workload to perform its function, often for authorization, accounting, or auditing. It may include user information, software and hardware information, or information about the request and its prior processing. Different pieces of context information may originate from different sources.

* Identity Proxy

The identity proxy is an intermediary that can inspect, replace or augment workload identity and security context information. A transparent network service such as a security gateway may act as an identity proxy, or the role can be implemented in a service performing explicit connection processing, such as an ingress gateway or a Content Delivery Network (CDN) service. The identity proxy MAY introduce additional context based on source identifier, communication properties and administrative policy. This context MAY be communicated as a transaction token {{I-D.ietf-oauth-transaction-tokens}}.

* Remote Attestation

The term "attestation", as defined in {{?RFC9683}}, refers to the process of generating and evaluating remote attestation Evidence. {{!RFC9334}} describes Evidence and the different communication patterns.

* Workload Identity Credential

A credential that contains a workload identifier ({{!WIMSE-ID=I-D.ietf-wimse-identifier}}) used for workload-to-workload authentication. The Workload Identity Certificate (WIC) and Workload Identity Token (WIT) defined in {{?WIMSE-CREDS=I-D.ietf-wimse-workload-creds}} bind a workload identifier to a public key. Their protocol bindings require proof of possession of the corresponding private key. Deployments may also accept legacy bearer credentials for interoperability, but these do not provide the same proof-of-possession guarantees. WITs always require proof of possession.

* Trust Domain

A trust domain is an administrative scope for workload identities. It defines an identity namespace and common policies for issuing and validating workload identity credentials. Credential issuance within a trust domain is managed by one or more authorized issuers. This scope is analogous to a Kerberos realm {{?RFC4120}}, but does not require a particular credential format or a single issuer. Trust domain identifiers are described in {{trust-domain}}.

* Issuer

An issuer issues workload identity credentials on behalf of one or more trust domains. It acts as a certification authority (CA) when issuing WICs. For WITs, it performs the JWT issuer role described in {{?RFC9901}}. Identifier assignment may be performed by a separate component under the trust domain's policy. An issuer may have multiple service instances and signing keys. Issuer signing keys are distinct from workload proof-of-possession keys.

* Token Service

A token service issues or exchanges authorization or security context tokens. In OAuth deployments, this function is performed by an authorization server. A component may act as both a token service and a workload identity issuer.

* Trust Anchor

A trust anchor is a public key and associated data that a relying party explicitly accepts as authoritative for validating credentials within a defined scope, following the terminology in {{?RFC9397}} and {{?RFC6024}}. It can be used to verify credential signatures directly or to validate certificate paths. The associated data constrains the purposes and identity namespaces for which the key is authoritative.

Trust anchor information can be represented by an X.509 certificate, including a self-signed certificate, or by a raw public key with associated metadata. A securely provisioned cryptographic hash can identify the trusted certificate or public key. In that case, the relying party also needs to obtain the complete certificate or public key and verify it against the hash before using it for signature validation. A key, certificate, or hash does not establish trust by itself: its use for the intended trust domain and purpose must be explicitly authorized through trusted configuration. A certificate's self-signature does not provide that authorization.

* Relying Party

A relying party is an entity that uses verified identity or security information to make an application decision. For workload authentication, it validates a workload identity credential and relies on the authenticated identity. In remote attestation, it uses Attestation Results from a verifier according to its appraisal policy, as described in {{RFC9334}}. This role can be performed by a workload, a gateway, or another component acting on behalf of a workload.

# Architecture

## Workload Identity Concepts {#whimsical-identity}

The workload identity architecture consists of three basic building blocks: a trust domain, a workload identifier, and identity credentials. These components provide the identity information used in authentication, authorization, and accounting processes. More complex workload identity constructs can be created from these basic building blocks.

### Trust Domain {#trust-domain}

Trust domain identifiers SHOULD be fully qualified domain names associated with the organization administering the trust domain. This naming convention helps to avoid identifier collisions; a domain name alone does not establish trust.

Each relying party MUST associate a trust domain with its authorized issuers, trust anchors, and constraints. For WITs, trusted public keys can be represented as a JSON Web Key (JWK) Set {{!RFC7517}}. For WICs, trust anchors are used to validate X.509 certification paths. A credential MUST be validated using only the trust anchors associated with the trust domain named in its workload identifier, subject to the configured scope of each issuer and key. This mapping and any updates MUST be obtained through a secure mechanism that provides authenticity, integrity, and freshness.

A single organization may define multiple trust domains for different purposes. Each trust domain must have a unique domain identifier. Workload identifiers are scoped within a trust domain as specified in {{Section 4.3 of WIMSE-ID}}. Identifiers that differ by trust domain represent distinct workload identities, even if the same underlying workload holds credentials in both domains.

#### Relationships among Trust Domains, Issuers, and Trust Anchors

The following table summarizes deployment variants supported by these definitions. Each association describes configured authority for credential validation. It does not by itself imply an independent cryptographic security boundary.

| Relationship | Cardinality | Deployment Rationale |
|--------------|-------------|---------------------------------|
| Trust domain to issuers | One or more | Separate issuers for WICs and WITs, or multiple issuers providing the same credential type. |
| Issuer to trust domains | One or more | A shared issuer serves several domains, with explicit authorization for each domain. |
| Issuer to signing keys | One or more | Key rotation or support for different signature algorithms. |
| Issuer to trust anchors | One or more | Credentials are validated using directly trusted issuer keys or certificate paths to CA trust anchors. |
| Trust domain to trust anchors | One or more | Multiple issuers, algorithm choices, or trust anchor rollover. |
| Trust anchor to trust domains | One or more | A common CA trust anchor serves several domains, subject to the applicable constraints. |
{: #trust-relationships title="Trust domain, issuer, and trust anchor relationships."}

For example, an issuer can use an elliptic-curve signing key and a separate Module-Lattice-Based Digital Signature Algorithm (ML-DSA) signing key. If both public keys are directly configured as trust anchors, that issuer is associated with two trust anchors, each with its own scope and algorithm policy.

Conversely, several issuing CAs can use distinct signing keys while their certificate paths terminate at a common trust anchor {{?RFC5280}}. Thus, multiple issuers or signing keys do not necessarily imply multiple trust anchors. The security consequences of shared keys and trust anchors are discussed in {{trust-domain-separation}}.

#### Signing Key Deployment Variants {#issuer-key-deployments}

The following cases distinguish the logical issuer, the trust domains it serves, and the entities authorized to use a signing key. Each variant offers operational benefits and introduces dependencies that deployments need to consider.

| Deployment variant | Benefits | Operational and security implications |
|--------------------|----------|---------------------------------------|
| Multiple instances of one issuer share a signing key. | Simplifies verification-key configuration for replicated issuer instances. | Requires coordinated key protection and rotation. Compromise of signing access at an instance can affect the issuer's full scope. |
| One issuer uses a signing key across multiple trust domains. | Reduces the number of signing keys and associated lifecycle operations. | Domain separation relies on issuance policies and relying-party checks. Key compromise affects every domain for which the key is trusted. |
| Separately administered issuers share a signing key. | Can ease migration or administrative handover by retaining an existing public verification key. | Requires coordination across administrators for key protection, rotation, and compromise response. The shared key alone cannot distinguish which issuer produced a signature. |
{: #signing-key-variants title="Benefits and implications of signing key deployment variants."}

Shared use of a signing key can be provided by a common signing service or by distributing the key to authorized instances. These approaches differ in where key material is held and how access is controlled. Sharing a CA trust anchor while retaining separate issuer signing keys is a further deployment choice, discussed in {{trust-domain-separation}}.

### Workload Identifier {#workload-identifier}

A workload identifier is a Uniform Resource Identifier (URI) that uniquely names a workload within a trust domain and is carried in workload identity credentials. Its format, syntax, comparison rules, and validation requirements are defined in {{WIMSE-ID}}.

The URI MUST include an authority identifying its trust domain. Trust domain naming is described in {{trust-domain}}. {{SPIFFE-ID}} provides an example of a workload identifier format.

Two credentials containing the same workload identifier value represent the same workload only when validated under the same trust domain and issuer trust configuration.

A workload identifier MAY represent a logical workload, a service implemented by one or more workloads, or a specific workload instance, depending on deployment policy. Relying parties MUST interpret the identifier according to the identity semantics defined by the issuing trust domain.

### Workload Identity Credentials {#workload-identity-credentials}

A WIC is an X.509 certificate and a WIT is a signed JWT. Both carry a workload identifier and a public key {{WIMSE-CREDS}}. The workload presents the credential and uses the corresponding private key to prove possession during authentication ({{workload-authentication}}).

Workloads obtain credentials from an issuer, directly or through a provisioning agent or platform component ({{credential-provisioning}}). Private keys may be generated and managed by the workload or these supporting components. They are stored securely to prevent unauthorized access. Key protection and lifecycle management are discussed in {{credential-theft}}.

## Workload Identity System Scenarios

### Basic Workload Identity Scenario {#basic-workload-identity-scenario}

~~~aasvg
                        +--------------------------------Trust Boundary---------------+
                        |                                                             |
                    .---+-.                                                           |
                    | .---+-.         +--------------+                                |
                    | |     |         |              |       (3)                      |
                    | |     |     +--->  Workload 1  +-------------------+            |
                    | |  G  |     |   |              |                   |            |
                    | |  a  |     |   +-------^------+                   |            |
                    | |  t  |     |           |                          |            |
+------------+      | |  e  |     |           | (1)                      |            |
|            | (2)  | |  w  | (3) |           |                          |            |
| App Client +----->| |  a  +-----+   +-------+-------+                  |            |
|            |      | |  y  |         |               |          +-------+------+     |
+------------+      | |     +-----+   |     Issuer    |    (1)   |              |     |
                    | |  S  | (4) |   |               +---------->   Workload 3 |     |
                    | |  e  |     |   |               |          |              |     |
                    | |  r  |     |   +-------+-------+          +-------+------+     |
                    | |  v  |     |           |                          |            |
                    | |  i  |     |           | (1)                      |            |
                    | |  c  |     |           |                          |            |
                    | |  e  |     |   +-------v------+                   |            |
                    | |     |     |   |              |      (4)          |            |
                    | |     |     +--->  Workload 2  +-------------------+            |
                    '-+     |         |              |                                |
                      '-+---'         +--------------+                                |
                        |                                                             |
                        |                                                             |
                        |                                                             |
                        +-------------------------------------------------------------+
~~~~
{: #arch-basic title="Basic example workload application system."}


{{arch-basic}} shows an application system within one trust domain: three workloads, a gateway, and an issuer. An external application client invokes the gateway.

* Workload

The workloads obtain credentials from the issuer (1) and use them to authenticate service requests.

* Gateway Service

The gateway routes external requests to the target workloads and may have multiple instances for availability. It MAY also implement identity proxy functionality, including authentication, token exchange, and token transformation.

* Issuer

The issuer provides WICs or WITs for workloads in this trust domain.

The numbered flows are:

* (1) Workload Identity Credential Distribution

Workloads obtain credentials from the issuer before using them. Provisioning mechanisms are discussed in {{credential-provisioning}}.

* (2) Application Client Requests

The client sends an API request to the gateway, typically over HTTPS. The gateway usually terminates TLS and may assist in authenticating the client before routing the request.

* (3) API Request to Workload 1

The gateway authenticates Workload 1, forwards the request, and supplies any relevant client information. Workload 1 may also authenticate the gateway. To service the request, Workload 1 calls Workload 3 using one of the authentication bindings in {{workload-authentication}}. Workload 3 authenticates Workload 1, authorizes the call, and records its identity for auditing. Workload 1 may also authenticate Workload 3.

* (4) API Request to Workload 2

For other requests, the gateway routes to Workload 2, which calls Workload 3 in the same way. Workload 3 may apply different authorization policies to Workloads 1 and 2 and rejects unauthorized requests.

### Context and Workload Identity

~~~aasvg
                  +------------------------Trust Boundary---------+
                  |                                               |
              .---+-.                     +----------------+      |
              | .---+-.                   |                |      |
              | |     +------------------>|    Context     |      |
              | |     |         (3)       |                |      |
              | |     |<------------------+    Service     |      |
              | |     |         (c)       |                |      |
              | |  G  |                   +----------------+      |
              | |  a  |                                           |
   (1)        | |  t  |                                           |
+-------+     | |  e  |      +------------+      +-------------+  |
|   App | (2) | |  w  | (4)  |            |  (5) |             |  |
| Client+---->| |  a  +----->| workload 1 +----->|  workload 2 |  |
+-------+ (a) | |  y  | (c)  |            |  (c) |             |  |
              | |     |      +------------+      +-------------+  |
              | |  S  |                                           |
              | |  e  |                                           |
              | |  r  |                                           |
              | |  v  |                                           |
              | |  i  |                                           |
              | |  c  |                                           |
              | |  e  |                                           |
              '-+     |                                           |
                '-+---'                                           |
                  |                                               |
                  |                                               |
                  |                                               |
                  +-----------------------------------------------+

~~~~
{: #arch-context title="Context example workload application system."}

{{arch-context}} extends the basic scenario with a context service. Credential provisioning and workload authentication details are omitted for simplicity.

* Context Service
The context service creates security context from authenticated client information and other request data. It may return data, a signed token, or a reference to stored context. In this example, it acts as a token service, exchanging the client's OAuth access token (a) for a context token (c), such as a transaction token.

* (1) Initial Credential Acquisition
The application client obtains a credential that it can present to the gateway service, such as an OAuth 2.0 access token. This may involve several steps, including authentication by an external identity provider. If the client already has a suitable credential, such as an access token or a TLS client certificate, it can skip this acquisition step.

* (2) Application Client Request
The application client makes a request to the gateway over HTTPS.  The client may be authenticated to the gateway through TLS client authentication (mutual TLS) or through a credential such as an access token obtained in step 1.

* (3) Establishing the Request Context
The gateway requests context (c) from the context service using the client's access token (a) and other relevant information. The context is intended for internal use and is normally not returned to the client.

* (4) Forwarding Request to Workload
The gateway forwards the request and context (c) to the target workload, which uses the context when authorizing the request. The external access token (a) is normally not forwarded. If policy requires context that is missing, the workload denies the request.

* (5) Calling Another Workload
Workload 1 includes the relevant context when calling Workload 2. Workload 2 may require both an authenticated workload identity and valid context before allowing the call.


### Cross-Domain Communication
~~~aasvg
                  +---------------------------Trust Boundary------+
                  |                                               |
              .---+-.                                             |
              | .---+-.                                           |
              | |     |                                           |
              | |  G  |                                           |
              | |  a  |                                           |
              | |  t  |                                           |
+-------+     | |  e  |      +------------+      +-------------+  |
|   App | (1) | |  w  | (2)  |            |  (4) |             |  |
| Client+---->| |  a  +----->| workload 1 +----->|  workload 2 |  |
+-------+ (a) | |  y  | (c)  |            |  (c) |             |  |
              | |     |      +-+----------+      +^----^----+--+  |
              | |  S  |        |                  |    |    |     |
              | |  e  |     (3)|            (5)(c)|(t) |    |     |
              | |  r  |        |                  |    |    |     |
              | |  v  |        |       +-+--------v+   |    |     |
              | |  i  |        |       |   Token   |   |    |     |
              | |  c  |        |       |           |   | (7)|(a)  |
              | |  e  |        |       |  Service  |   |    |     |
              '-+     |        |       +-----------+   |    |     |
                '-+---'        |                       |    |     |
                  |            |                       |    |     |
                  |            |                 (6)(t)|(a) |     |
                  |            |                       |    |     |
                  +------------+-----------------------+----+-----+
                               |                       |    |
                               |             +-+-------V-+  |
                               |             |  External |  |
                               |             |   Token   |  |
                               |             |  Service  |  |
                               |             +-----------+  |
                               |                            |
                     +---------v------+     +---------------v-+
                     | Infrastructure |     |     External    |
                     |                |     |                 |
                     |    Service     |     |     Service     |
                     +----------------+     +-----------------+
~~~~
{: #arch-cross title="External request workload application system."}

{{arch-cross}} adds access to infrastructure services and other trust domains. Steps 5-7 follow the cross-domain pattern in {{?I-D.ietf-oauth-identity-chaining}}, with token services performing the authorization-server functions. As before, (c) denotes security context. Workload authentication and provisioning are omitted from the diagram.

* Token Service - Issues a token (t) using the workload's authenticated identity and context (c). The token can be presented to the external token service.

* External Token Service - Exchanges (t) for an access token (a) accepted by the external service in its trust domain.

* Infrastructure Service - Managed by an infrastructure provider and may have different access requirements from the application workloads.

* External Service - A service in another trust domain with its own access policy.

Some example interactions in this scenario:

* (1) The application client sends a request to the gateway using its access token (a).

* (2) The gateway forwards the request and context (c) to Workload 1.

* (3) Workload 1 authenticates to the infrastructure service with its workload identity credential under an established trust relationship.

* (4) Workload 1 calls Workload 2 with context (c).

* (5) To access the external service, Workload 2 authenticates to the token service with its workload identity credential and supplies context (c) to obtain token (t).

* (6) Workload 2 presents (t) to the external token service to obtain access token (a).

* (7) Workload 2 presents (a) to the external service.

Where the trust relationship permits it, a workload may also use its workload identity credential directly with an external token service to obtain an access token.

## Message Flow and Policy Enforcement in the Basic Scenario {#arch-basic-ops}

{{arch-basic-ops-fig}} expands one workload-to-workload hop from {{arch-basic}}, showing provisioning, authentication, authorization, and response.

A Policy Enforcement Point (PEP) enforces authorization decisions on the callee's request path. It may consult a Policy Decision Point (PDP) for policy evaluation. These roles may be embedded in a workload or deployed in a gateway, sidecar, or separate service. Policy languages and PEP-to-PDP protocols are outside this document's scope.

The diagram is illustrative rather than normative.

~~~aasvg
+------------+               +------------+
|            |      (2)      |            |
|            |<=============>|            |
|            |               |            |
| Workload A |      (3)      | Workload B |
|            |==============>|            |
|            |               |            |
|            |      (5)      |   +--------+
|            |<==============|   |  PEP   |
+------------+               +---+--------+
      ^                        ^     ^
      |            (1)         |     |
  (1) | +----------------------+     | (4)
      | |                            |
      v v                            v
+------------+               +------------+
|            |               |            |
|   Issuer   |               |    PDP     |
|            |               | (optional) |
|            |               |            |
+------------+               +------------+
~~~
{: #arch-basic-ops-fig title="Logical message flow and policy points (drill-down of the basic scenario)."}

The high-level message flow is as follows:

1. Workloads A and B obtain credentials from the issuer using the mechanisms in {{credential-provisioning}}. Credential lifetime and renewal depend on the deployment.
2. A transport connection is set up for the call. It may use mutual TLS with WICs or another transport security mechanism.
3. Workload A sends a request to Workload B, optionally including a WIT and proof of possession as described in {{workload-authentication}}. Workload B authenticates Workload A.
4. Workload B authorizes the request. The PEP enforces the decision, optionally consulting a PDP. Policy details and message-level authorization formats are out of scope for this architecture.
5. Workload B returns a response to Workload A, which may be an error or success.

Depending on the protocol binding, peer authentication may occur during step (2) at the transport layer, during step (3) at the application layer, or both, as in the other scenarios in this section.

## Workload Identity Use Cases

### Bootstrapping Workload Identifiers and Credentials {#credential-provisioning}

Workloads obtain WICs or WITs from an issuer before using them for authentication. The following provisioning patterns are illustrative and may be combined. Infrastructure-provided bootstrap credentials and attestation can reduce reliance on long-lived secrets. Further delivery patterns are described in {{Section 3 of ?I-D.ietf-wimse-workload-identity-practices}}.

* Direct Provisioning - A workload may receive its WIMSE credential when it is initially created. A deployment-specific mechanism delivers the credential and its associated private key, or enables access to an authorized signing component. The workload can then use the credential without a separate enrollment exchange.

* Bootstrap Credentials - A workload may receive bootstrap credentials specific to its deployment mechanism and use them to obtain a WIMSE credential and access to its associated private key. Alternatively, the workload may generate a key pair and use the bootstrap credential to authorize enrollment of the public key in a WIC or WIT. {{I-D.ietf-wimse-workload-identity-practices}} describes some ways workloads can be initially provisioned with bootstrap credentials.

* Attestation - Attestation can inform a provisioning decision in addition to, or instead of, other bootstrap mechanisms. An attester generates Evidence about itself or its execution environment. A verifier appraises the Evidence and produces Attestation Results, which a relying party uses when deciding whether to authorize provisioning {{RFC9334}}. These roles may be co-located. This architecture does not require a particular evidence format, implementation, or attestation protocol. See {{SPIRE}} for an example of node and workload attestation supporting credential provisioning.

* Provisioning Agent - A node-local provisioning agent can obtain credentials from an issuer on behalf of workloads. It may deliver credentials and associated key material, provide access to a signing component, or relay an encrypted key that only the workload can decrypt. The example below illustrates one such arrangement.

#### Provisioning Agent Example {#agent-assisted-provisioning-example}

In {{arch-fig}}, a provisioning agent supplies attestation Evidence and other information to an issuer. The issuer evaluates this information itself or relies on a verifier to make its provisioning decision. The agent then delivers the issued credentials to the workloads it serves.

~~~aasvg
  +-----------------+
  |     Issuer      |
  |                 |
  | +-------------+ |
  | | Attestation | |
  | +-------------+ |
  +---------+-------+
          ^ |                              . .
          | | Identity                     | | Workload
          | | Credentials                  | |    to
          | |                              | | Workload
          | |                              | | Communication
  +-------+-+------------------------------+-+-----------+
  |       | |                              v V           |
  |       | v                         +----------------+ |
  |  +----+----------+              +-+--------------+ | |
  |  | Provisioning  |              | Workloads      | | |
  |  | Agent        <+--------------+>               | | |
  |  |            ^  |  Identity    |  ^             +-+ |
  |  +------------+--+  Credentials +--+-------------+   |
  |               |                    |                 |
  |               |                    | Identity        |
  |   Attestation |                    | Credentials     |
  |               v                    v                 |
  +------------------------------------------------------+
  | Host Operating System and Hardware                   |
  +------------------------------------------------------+
~~~~
{: #arch-fig title="Host Software Layering in a Workload Identity Architecture."}


Provisioning agents or platform components may deliver credentials through:

* File System - Credential files made available to the workload.
* Local API - A local domain socket (for example, a SPIFFE or QEMU guest agent API) or a platform network API (for example, a cloud provider metadata service).
* Environment Variables - Credentials injected into the workload's environment.

### Workload and Service Authentication {#workload-authentication}

Workload authentication establishes the peer identity used for authorization ({{authorization}}) and auditing ({{audit-trails}}). The trust domain determines whether that identity represents a service, workload, or workload instance, as described in {{workload-identifier}}.

Once credentials are issued, the protocol binding determines how they are presented and how possession of the associated private key is demonstrated:

| Mechanism | Credential | Proof of possession |
|-----------|------------|---------------------|
| Mutual TLS {{?WIMSE-MTLS=I-D.ietf-wimse-mutual-tls}} | WIC | TLS handshake. |
| Workload Proof Token {{?WIMSE-WPT=I-D.ietf-wimse-wpt}} | WIT | A JWT signed with the workload's private key. |
| HTTP Message Signatures {{?WIMSE-HTTPSIG=I-D.ietf-wimse-http-signature}} | WIT | An HTTP message signature using the workload's private key. |
{: #workload-authentication-bindings title="Workload credentials and authentication protocol bindings."}

Both HTTP-based application-layer mechanisms convey the WIT in the Workload-Identity-Token header field. The WPT binding carries the proof in the Authorization field using the WPT scheme. The HTTP Message Signature binding carries the proof in the Signature-Input and Signature fields. In each case, the recipient validates the WIT and the accompanying proof according to the applicable specification.

Transport- and application-layer authentication can be combined as described in {{layered-workload-authentication}}. {{arch-chain}} illustrates communication across trust domains and through intermediary workloads.

~~~aasvg
  +-----------------+
  |     Workload    |
  |    (external)   |
  |       ^         |
  +-------+---------+
          |
          |
  +-------+-------------------------Trust Boundary---------------+
  |       |                                                      |
  |       |                                                      |
  |  +----+------+ Hop-by-  +-----------+ Hop-by-  +-----------+ |
  |  |    v      | Hop      |           | Hop      |           | |
  |  | Workload  | Security | Workload  | Security | Workload  | |
  |  |          <+----------+>         <+----------+>          | |
  |  |           |          |           |          |           | |
  |  |           |          |           |          |           | |
  |  |    O<-----+----------+-----------+----------+---->O     | |
  |  +-----------+   E2E    +-----------+   E2E    +-----------+ |
  |                                                              |
  |                                                              |
  +--------------------------------------------------------------+
~~~~
{: #arch-chain title="Workload-to-Workload Communication."}

### Layered Workload Authentication {#layered-workload-authentication}

Some deployments authenticate at both the transport and application layers. In {{arch-layered-auth}}, Workload A uses a WIC on the TLS hop to a proxy and a WIT with proof of possession to authenticate to Workload B. Because the proxy terminates A's TLS connection, B uses the application-layer proof to identify A. The two layers may use different peer identities, trust anchors, and authorization policies.

~~~aasvg
                 Transport-layer              Transport-layer
                 authentication               authentication
 +------------+   (e.g., TLS)    +---------+   (e.g., TLS)    +------------+
 |            |<---------------->|         |<---------------->|            |
 | Workload A |                  |  Proxy  |                  | Workload B |
 |            |                  | Gateway |                  |            |
 +-----+------+                  +---------+                  +------+-----+
       ^                                                             ^
       |                                                             |
       |        Application-layer authentication and context         |
       +=============================================================+
                    (e.g., WIT with proof of possession)
~~~~
{: #arch-layered-auth title="Layered workload authentication through an intermediary."}

Forwarding application-layer authentication through an intermediary requires the proof to identify the intended recipient and its protected data to remain verifiable. A WPT binds its associated WIT and accompanying authorization or context tokens as specified in {{WIMSE-WPT}}, but does not provide general HTTP message integrity. HTTP Message Signatures protect the components specified by their profile and changes to a covered component invalidate the signature {{WIMSE-HTTPSIG}}. If an intermediary uses its own credential and produces a new proof, the destination authenticates that intermediary. Any asserted relationship to the original caller requires separate validation under the applicable policy.

Deployment policy defines when transport- and application-layer identities must match, when one entity may act on behalf of another, and whether intermediaries may replace or augment identity or context information. Such intermediary behavior needs to be explicit and auditable. A destination workload MUST NOT assume that a transport-layer identity authenticated by an intermediary is the same as the application-layer caller identity unless that relationship is explicitly established by protocol or deployment policy.

Audit records for layered authentication SHOULD record both the transport-layer identity and the application-layer identity when both are available and relevant. This distinguishes the channel peer from the application-layer caller.

### Service Authorization {#authorization}

Authorization determines whether the authenticated workload identity may perform the requested operation on a resource.

Authorization policy may use the authenticated workload identity alone or combine it with security context from trusted sources, such as gateways or identity proxies. Relevant context may include end-user attributes, trust domain policies, or deployment-specific metadata (e.g., environment, service role, workload instance).

Authorization decisions typically include:

* Validating the integrity and provenance of any security context used in the decision.
* Ensuring the authenticated identity has the correct role and attributes to access the requested API or resource.
* Applying fine-grained policy rules, which may include path, method, action type, and contextual constraints (e.g., geographic location, time of day).

Authorization checks may also incorporate delegation and impersonation semantics, as described in {{delegation}}, where upstream workloads are authorized to act on behalf of end-users or other services, within the scope of their issued credentials and policy.

Authorization is enforced by the resource-owning workload or by a trusted PEP acting on its behalf. Policy evaluation may be delegated to a PDP, including a component in the caller's environment, when the resource owner trusts that component. The resource owner needs to ensure that a remote decision is authentic, applies to the current request, and is enforced on every access path. Caller-side checks can provide additional controls, but knowledge of policy and context alone does not authorize a caller to grant itself access.

### Audit Trails {#audit-trails}

Auditability is a critical requirement in systems that rely on workload identities and security context. Each authenticated request MUST leave a verifiable and inspectable trace regardless of authentication and authorization decision.

Audit trails are typically generated at multiple points:

* Gateway Services: Log incoming client requests, their authenticated identities and relevant context.
* Workloads: Log authenticated peer identities, security context attributes, requested resources, and authorization outcomes.
* Issuers and Token Services: Log issuance and validation events for workload identity credentials and authorization or context tokens, respectively.

Audit records may include:

* Timestamp of the request
* Source workload identifier
* Target workload identifier
* Authentication method used
* Decision outcome (authorized/denied)
* Security context claims
* Delegation/impersonation metadata (if present)

To avoid inadvertent disclosure of sensitive information, workloads and services generating audit logs MUST NOT log secrets such as bearer tokens, private keys, or passwords. Implementations MUST NOT log raw, replayable proofs of possession by default. If logging of credential-related data is necessary for diagnostic or policy purposes, these values MUST be redacted, hashed, or otherwise sanitised to prevent misuse.

WIMSE systems SHOULD ensure audit logs are tamper-evident and securely stored. Logs may be forwarded to centralized security information and event management (SIEM) systems to enable compliance, threat detection, and incident response.

### Security Context Establishment and Propagation {#seccontext}

The context service in {{arch-context}} illustrates how workloads obtain and forward security context. A workload may augment the context with information about processing it performs. Each recipient evaluates the context's integrity and provenance before using it for authorization or auditing.

Context tokens may identify their source, intended recipients, or workloads allowed to present them. Binding a token to an authenticated workload or request lets the recipient check that it is being used in the intended context. Application-layer proofs can cover context tokens as specified in {{workload-authentication}}.

### Delegation and Impersonation {#delegation}

A workload may authenticate to a token service and obtain a token authorizing it to act on behalf of another principal, for example through OAuth Token Exchange {{?OAUTH-TOKEN-XCHANGE=RFC8693}}. The token carries delegated authority or impersonation context and may be bound to the workload identity ({{binding}}).

### Accessing Resources in Other Trust Domains

The token exchange in {{arch-cross}} illustrates how a workload obtains authorization to access another trust domain. The destination may also rely on services in further domains, using the delegation mechanisms described in {{delegation}}.

WICs and WITs bind the presenting workload's identifier to a public key under its trust domain's trust configuration. They do not define the subjects or actor chains of separate delegation or impersonation tokens. A receiving workload validates each such token according to its format, the trust configuration for that token's issuer, and applicable authorization policy, and checks any binding to the authenticated workload. The trust relationships for workload authentication and delegated authorization may differ.

### Asynchronous and Batch Requests

Source workloads may submit asynchronous or batch requests that are executed after their original identity credentials expire. The executing workload obtains or renews its own WIC or WIT from an issuer. Where continued delegated authority is required, a token service can issue a suitably scoped authorization or context token under deployment policy. That token may identify the originating principal as its subject and the executing workload as an actor, if its format supports those semantics. Token validation and any binding to the executing workload are separate from WIC or WIT validation. Renewing an identity credential does not renew delegated authority. Deployments need to define how authorization lifetime, renewal, and revocation affect queued work.

### Cross-boundary Workload Identity

Cross-domain authentication and token exchange are illustrated in {{arch-cross}}. This section describes how identity proxies can limit disclosure of internal identities.

#### Egress Identity Generalization

Workload identifiers can reveal internal topology, such as instance names or deployment details. A workload may use a generalized service identity when communicating across trust domains to limit this disclosure.

An identity proxy can authenticate the internal workload and present the generalized identity externally when authorized by policy. The proxy uses a credential for that identity from an authorized issuer. The external relying party validates the credential and any proof required by its protocol binding.

#### Inbound Gateway Identity Validation

An inbound gateway should validate external workload credentials as a relying party. It may forward verifiable caller authentication to the destination or authenticate onward using its own credential, as described in {{layered-workload-authentication}}.

### AI and ML-Based Intermediaries

Emerging agentic AI systems and other ML-based intermediaries introduce new considerations for workload identity and security context propagation. These systems may act on behalf of an upstream principal (such as a user or service), or on their own behalf, and invoke downstream workloads as part of multi-step workflows. Agent identity and authorization scenarios are discussed in {{?WIMSE-AIMS=I-D.ietf-wimse-aims}}.

When an AI intermediary acts on behalf of an upstream principal, it is a delegated workload (see {{delegation}}) and operates within the scope of that delegation. When invoking downstream workloads, the agent SHOULD propagate only the security context required and authorized for the downstream request. Where necessary, an authorized token service can translate or down-scope that context.

In some cases, AI systems may generate requests that are not attributable to a specific upstream principal. Such autonomous actions MUST be clearly distinguished from delegated ones, for example by using separate workload identities or token scopes. Because AI intermediaries may chain requests across multiple services, there is an elevated risk of privilege escalation if security context is propagated beyond the intended trust domain. Mechanisms such as cryptographic binding of delegation tokens or attestation of intermediary behavior can help mitigate these risks.

A further consideration arises when AI agents interact with other AI agents. In these cases, each agent may act both as a delegated workload and as a delegator, creating multi-hop delegation chains. To avoid ambiguity, each hop in the chain MUST explicitly scope and re-bind the security context so that downstream services can reliably evaluate provenance and authorization boundaries. Without such controls, there is a risk that a chain of AI-to-AI interactions could unintentionally extend authority far beyond what was originally granted.

~~~aasvg
   +---------+        +-------------+        +-------------+        +-------------+
   |  User / +------->|  AI Agent   +------->|  AI Agent   +------->|  Workload   |
   | Service |        |  (Agent A)  |        |  (Agent B)  |        |  Downstream |
   +---------+        +-------------+        +-------------+        +-------------+
        |                    |                      |                      |
        |     Initial        |  Delegated / Scoped  |  Delegated / Scoped  |
        | Security Context   |   Security context   |   Security context   |
        +------------------->+--------------------->+--------------------->|
~~~~
{: #arch-ai title="AI agent communication"}

# Security Considerations

## Traffic Interception

TLS protects traffic between adjacent TLS endpoints. A terminating intermediary, such as a gateway or proxy, can access the plaintext and may modify it. Deployments need to assess confidentiality, integrity, and preservation of authenticated identity at each termination point. Application-layer authentication across such intermediaries is discussed in {{layered-workload-authentication}}.

## Information Disclosure

Logs and other observability systems can expose tokens, workload identifiers, user information, and security context. Access to these records may be less restricted than access to the originating workload. Disclosure can enable replay, reveal deployment details, or expose personal information. Logging controls are discussed in {{audit-trails}}.

## Workload Identity Binding to Tokens {#binding}

Binding an authorization or context token to a workload can restrict who may present or consume it ({{seccontext}}). This mitigates token theft only when the relying party validates the token and checks its binding to the authenticated identity or request.

## Credential Theft {#credential-theft}

A stolen bearer credential can be replayed directly. Capturing a WIC or WIT alone does not provide the corresponding private key, but capturing a usable proof of possession may allow replay within its validity window. The protocol binding determines proof lifetime and replay controls.

Private keys need sufficient entropy and protection against unauthorized access. Credential and key lifetimes depend on workload lifetime, deployment constraints, sensitivity, and the key-protection mechanism. Generating a new key pair for every request is not generally required. Deployments SHOULD use bounded credential lifetimes and automated renewal so that credentials and associated key pairs are periodically replaced. Renewal needs to support long-running workload instances without relying on long-lived static secrets. Following suspected compromise, deployments should revoke affected credentials or stop accepting them according to the available mechanisms and local policy.

## Trust Domain Separation {#trust-domain-separation}

Distinct trust domain names or issuer identifiers do not provide cryptographic isolation when a signing key is trusted across domains ({{issuer-key-deployments}}). Compromise of that key permits forgery within its full configured scope. Selecting trust anchors according to the trust domain does not prevent this when the same key is authorized for each domain.

A common X.509 CA trust anchor does not necessarily imply shared issuer signing keys. Separate intermediate CAs with domain-specific constraints, such as applicable name constraints, can restrict their issuing authority if relying parties enforce those constraints {{?RFC5280}}. However, the common root authority remains a shared trust dependency, and compromise of its private key can affect every domain that trusts it.

The choice among these variants depends on the required availability, administrative independence, and key-management effort. Deployments requiring isolation against compromise of another domain's issuing key need separate signing keys and enforceable domain-specific limits on issuing authority. Their choice of trust anchors also determines which root authorities remain shared dependencies.

## Authentication and Authorization

Relying parties need to distinguish authentication from authorization. WIC- and WIT-based authentication requires a valid issuer-signed credential and proof of possession. Legacy bearer authentication does not provide that proof. A valid credential, including one from the relying party's own trust domain, does not authorize access by itself. The resource owner still applies the policy described in {{authorization}}.

## Workload Compromise

Even the most well-designed and implemented workloads may contain security flaws that allow an attacker to gain limited or full compromise. For example, a server side request forgery may result in the ability for an attacker to force the workload to make requests of other parts of a system even though the rest of the workload functionality may be unaffected. An attacker with this advantage may be able to utilize privileges of the compromised workload to attack other parts of the system. Therefore it is important that communicating workloads apply the principle of least privilege through security controls such as authorization.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

Todo: Add your name here.

# Changes since draft -05
{:numbered="false"}

* Update to gateway service definition and diagram
* alignment of cross-domain scenario with OAUTH cross-domain chaining
* rework of authentication section
* added audit section
* added AI use case

# Changes since draft -06
{:numbered="false"}

* Separated Workload from Workload Instance
* Moved workload identifier definition to a separate draft
* Update credential provisioning section
* Add {{arch-basic-ops}}: message flow and PEP/PDP drill-down of the basic scenario, before use cases
