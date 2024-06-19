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
    organization: Venafi
    email: joe@salowey.net
 -
    ins: Y. Rosomakho
    name: Yaroslav Rosomakho
    organization: Zscaler
    email: yaroslavros@gmail.com
 -
    ins: H. Tschofenig
    name: Hannes Tschofenig
    organization:
    email: hannes.tschofenig@gmx.net

normative:

informative:

--- abstract

The increasing prevalence of cloud computing and micro service architectures has led to the rise of complex software functions being built and deployed as workloads, where a workload is defined as a running instance of software executing for a specific purpose.  This document discusses an architecture for designing and standardizing protocols and payloads for conveying workload identity and security context information.

--- middle

# Introduction

The increasing prevalence of cloud computing and micro service architectures has led to the rise of complex software functions being built and deployed as systems composed of workloads, where a workload is defined as a running instance of software executing for a specific purpose.

Workloads need to be provisioned with an identity when they are started. Often, additional information needs to be provided, such as trust anchors and security context details. Workload identity credential is used to authenticate communications between workloads. Workloads make use of identity information and additional context information to perform authentication and authorization.

This architecture considers two ways to express identity information: X.509 certificates often used in the TLS layer and JSON Web Tokens (JWTs) used at the application layer. Collectively, these are referred to as WIMSE tokens. The applicability of given token format depends on application and security context and will be explored in later sections.

Once the workload is started and has obtained identity information, it can start performing its functions. Once the workload is invoked it may require interaction with other workloads. An example of such interaction is shown in {{?I-D.ietf-oauth-transaction-tokens}} where an externally-facing endpoint is invoked using conventional authorization mechanism, such as an OAuth 2.0 access token. The interaction with other workload may require the security context associated with the authorization to be passed along the call chain.

In the rest of the document we describe terminology and use cases, discuss details of the architecture, and discuss threats.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the following terms:

* Workload

A workload is a running instance of software executing for a specific purpose. Workload typically interacts with other parts of a larger system. A workload may exist for a very short durations of time (fraction of a second) and run for a specific purpose such as to provide a response to an API request. Other kinds of workloads may execute for a very long duration, such as months or years. Examples include database services and machine learning training jobs.

* Security Context

A security context provides information needed for a workload to perform its function. This information is often used for authorization, accounting and auditing purposes and often contains information about the request being made. Some examples include user information, software and hardware information or information about what processing has already happened for the request. Different pieces of context information may originate from different sources.

* Identity Proxy

Identity proxy is an intermediary that can inspect, replace or augment workload identity and security context information. Identity proxy can be a capability of a transparent network service, such as a security gateway, or it can be implemented in a service performing explicit connection processing, such as an ingress gateway or a Content Delivery Network (CDN) service.

* Attestation

Attestation is the function through which a task verifies the identity of a separate Workload. (TBD: sync definition with reference RaTS architecture)

# Architecture

## Workload Identity

### Bootstrapping Workload Identity

A workload needs to obtain its identity early in its lifecycle. This identity is sometimes referred to as the "bottom turtle" on which further identity and security context is built.

Identity bootstrapping often utilizes identity information provisioned through mechanisms specific to hosting platforms and orchestration services. This initial bootstrapping information is used to obtain specific identity credentials for a workload. This process may use attestation  to ensure the workload receives correct identity credentials. An example of a bootstrapping process follows.

{{arch-fig}} provides an example of software layering at a host running workloads. During startup, workloads bootstrap their identity with the help of an agent. The agent may be associated with one or more workloads to help ensure that workloads are provisioned with the correct identity. The agent provides attestation evidence and other relevant information to a server. After obtaining identity credentials from the Server it passes them to the workload. The server validates this information and provides the agent with identity credentials for the workloads it is associated with. The server can use a variety of internal and external means to validate the request against policy.

~~~aasvg
  +-----------------+
  |     Server      |
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
  |  | Agent         |              | Workloads      | | |
  |  |              <+--------------+>               | | |
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

How the workload obtains its identity credentials and interacts with the agent is subject to different implementations. Some common mechanisms for obtaining this initial identity include:

* File System - in this mechanism the identity credential is provisioned to the workload via the filesystem.
* Local API - the identity credential is provided through an API, such as a local domain socket (for example SPIFFE or QEMU guest agent) or network API (for example Cloud Provider Metadata Server).
* Environment Variables - identity credential may also be injected into workloads using operating system environment variables.

### Identity Credentials

The Agent provisions the identity credentials to the workload. These credentials are represented in form of JWT tokens and/or X.509 certificates.

JWT bearer tokens are presented to another party as a proof of identity. They are signed to prevent forgery, however since these credentials are often not bound to other information it is possible that they could be stolen and reused elsewhere. To reduce some of these risks, bearer tokens may have a short lifespan. Alternatively, sender constrained tokens can be used such as TLS session binding.

X.509 certificate credentials consist of two parts:

- a certificate is a signed data structure that contains a public key and identity information
- a corresponding private key

The certificate is presented during authentication, however the private key is kept secret and only used in cryptographic computation to prove that the presenter has access to the private key that corresponds to the public key in the certificate.

## Workload Identity Use Cases

### Basic Service Authentication and Authorization

One of the most basic use cases for workload identity is authentication of one workload to another, such as in the case where one service is making a request to another service as part of a larger, more complex application. Following authentication, the request to the service offered by the workload it needs to be authorized. Even in this simple case the identity of a workload is often composed of many attributes such as:

* Trigger Information
* Service Name
* Instance Name
* Role
* Environment
* Trust Domain
* Software Attestation
* Hardware Attestation

These attributes are used for various purposes such as:

* ensuring the request is made to the correct service or service instance
* authorizing access to APIs and resources
* providing an audit trail of requests within a system
* providing context for other decisions made within a service

There are several methods defined to perform this authentication.  Some of the most common include:

* TLS authentication of the server using X.509 certificates and bearer token, encoded as JWTs.
* Mutual TLS authentication using X.509 certificate for both client and server.
* TLS authentication of the server and HTTP request signing using a secret key.

{{arch-chain}} illustrates the communication between different workloads. Two aspects are important
to highlight: First, there is a need to consider the interaction with workloads that are external
to the trust domain (sometimes called cross-domain). Second, the interaction does
not only occur between workloads that directly interact with each other but instead may also
take place across intermediate workloads (in an end-to-end style).

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

### Security Context Establishment and Propagation

In a typical system of workloads additional information is needed in order for the workload to perform its function. For example, it is common for a workload to require information about a user or other entity that originated the request. Other types of information may include information about the hardware or software that the workload is running or information about what processing and validation has already been done to the request. This type of information is part of the security context that the workload uses during authorization, accounting and auditing. This context is propagated and possibly augmented from workload to workload using tokens. Workload identity comes into play to ensure that the information in the context can only be used by an authorized workload and that the context information originated from an authorized workload.

One example of security context establishment and propagation is as follows:

~~~aasvg
┌───────────────────────────────────────────────────────────┐
│                                                           │
│       ┌───────────────────┐     ┌──────────────────┐      │
│       │  -Identity 1      │     │ -Identity 1 + 2  │      │
│       │  -Security Context│     │ -Security Context│      │
│       └──────────┬────────┘     └────┬─────────────┘      │
│                  │                   │                    │
│    ┌──────────┐  ▼   ┌──────────┐    ▼ ┌──────────┐       │
│    │ Workload1├──────► Workload2├──────► Workload3│       │
│    └─────▲────┘      └──────────┘      └──────────┘       │
│          │                                                │
│          │Attestation                                     │
│          │                                                │
├──────────▼────────────────────────────────────────────────┤
│                                                           │
│    Platform: Host Operating System/Hardware Software      │
└───────────────────────────────────────────────────────────┘
~~~~
{: #arch-chain title="Security context establishment and propagation"}

1. Workload 1 requests to load the LLM(Large Language Model) within Workload 2; Workload 2 requires that Workload 1 has hardware-based security to prevent model leakage. The hardware-based security context information of Workload 1 can be obtained by running a remote attestation protocol. Workload 1 provides its Identity and the security context information to Workload 2 to apply for loading the LLM.

2. Workload 2 verifies the Identity and corresponding security context information provided by Workload 1 to confirm whether the model can be provided to Workload 1.

3. Workload 1 also requires the LLM on Workload 2 to use the vertical industry domain-specific data on Workload 3 for model fine-tuning training.

4. Workload 2 provides its Identity, Workload 1’s Identity, and its hardware-based security context information to Workload 3.

5. Workload 3 uses this information for authentication and, upon passing, provides the industry domain-specific data to Workload 2.

6. After fine-tuning training with the industry domain-specific data provided by Workload 3, Workload 2 provides the Model to Workload 1.



### Delegation and Impersonation

TBD.

### Asynchronous and Batch Requests

TBD.

### Cross-boundary Workload Identity

As workloads often need to communicate across trust boundaries, extra care needs to be taken when it comes to identity communication to ensure scalability and privacy. (TODO: align with OAuth cross domain identity and authorization)

#### Egress Identity Generalization

A workload communicating with a service, or another workload located outside the trust boundary may need to provide modified identity information. Detailed identity of internal workload originating the communication is relevant inside the trust boundary but could be excessive for the outside world and expose potentially sensitive internal topology information.

A security gateway at the edge of a trust boundary can be used to validate identity information of the workload, perform context specific authorization of the transaction and replace workload specific identity with a generalized one for a given trust domain.

#### Inbound Gateway Identity Validation

Inbound security gateway is a common design pattern for service protection. This functionality is often found in CDN services, API gateways, load balancers, Web Application Firewalls (WAFs) and other security solutions. Workload identity verification of inbound requests should be performed as a part of these security services. After validation of workload identity, the gateway may either leave it unmodified or replace it with its own identity to be validated by the destination.

# Security Considerations

## Traffic Interception

Workloads communicating with applications may face different threats to traffic interception in different deployments. In many deployments security controls are deployed for internal communications at lower layers to reduce the risk of traffic observation and modification for network communications. When a security layer, such as TLS, is deployed in these environments. TLS may be terminated in various places, including the workload itself, and in various middleware devices, such as load balancers, gateways, proxies, and firewalls. Therefore, protection is provided only between each adjacent pair of TLS endpoints. There are no guarantees of confidentiality, integrity and correct identity passthrough in those middleware devices and services.

## Information Disclosure

Observation and interception of network traffic is not the only means of disclosure in these systems. Other vectors of information leakage is through disclosure in log files and other observability and troubleshooting mechanisms. For example, an application may log the contents of HTTP headers containing JWT bearer tokens. The information in these logs may be made available to other systems with less stringent access controls, which may result in this token falling into an attackers hands who then uses it to compromise a system. This creates privacy risks and potential surface for reconnaissance attacks. If observed tokens can be reused, this also may allow attackers to impersonate workloads.

## Workload Compromise

Even the most well-designed and implemented workloads may contain security flaws that allow an attacker to gain limited or full compromise. For example, a server side request forgery may result in the ability for an attacker to force the workload to make requests of other parts of a system even though the rest of the workload functionality may be unaffected. An attacker with this advantage may be able to utilize privileges of the compromised workload to attack other parts of the system. Therefore it is important that communicating workloads apply the principle of least privilege through security controls such as authorization.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

Todo: Add your name here.
