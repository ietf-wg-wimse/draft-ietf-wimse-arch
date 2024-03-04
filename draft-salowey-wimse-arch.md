---
title: "Workload Identity in a Multi System Environment (WIMSE) Architecture"
abbrev: "WIMSE Architecture"
docname: draft-salowey-wimse-arch-latest
category: info
submissionType: IETF

ipr: trust200902
area: General
workgroup: wimse
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

The increasing prevalence of cloud computing and micro service architectures has led to the rise of complex software functions being built and deployed as workloads, where a workload is defined as a running instance of software executing for a specific purpose.

Workloads need to be provisioned with a unique identity when they are started. This information  their security context needs to be passed along the call chain. This architecture document discusses the motivation for designing and standardizing protocols and payloads for conveying identity and security context information.

--- middle

# Introduction

The increasing prevalence of cloud computing and micro service architectures has led to the rise of complex software functions being built and deployed as workloads, where a workload is defined as a running instance of software executing for a specific purpose.

Workloads need to be provisioned with a unique identity when they are started. Often, also other information needs to be provided, such as trust anchors. We use the term "identity" in a generic way to express that the information may vary with deployments since workloads run applications and some of these applications may require X.509 cerificates (along with the private key), or JSON Web Tokens (JWTs) acting as bearer tokens at the application layer, or both.

{{arch-fig}} shows the software layering at a host running workloads. As the workloads get started, they get their identity provisioned with the help of an agent. The agent is responsible for interacting with a server that ensures workloads in set of hosts are managed conviently and identity provisioned to workloads are associated with the expected authorization privileges. The server manages the lifecycle of the workloads with the help of the agent. The agent may also need to request attestation information about the hardware, lower layer software/firmware, and characteristics of the workload before the server identity information can be obtained.

How the workload obtains identity information and interacts with the agent is subject to different implementations, as described in this document. A few variants (such as environmet variables or domain sockets) have been used in deployments today.

~~~
   +---------------+
   |    Server     |
   |               |
   +---------------+
           ^|
           ||
           ||
           || Identity
           || Information                   ..
           ||                               ||
           ||                               || Workload
           ||                               ||    to
           ||                               || Workload
           ||                               || Communication
  +--------||-------------------------------vv-----------+
  |        |v                        +-----------------+ |
  |  +---------------+              +-----------------+| |
  |  | Agent         |              | Workload        || |
  |  |               |              |                 || |
  |  |              <...............>                 || |
  |  |            ^  |  Identity    | ^               |+ |
  |  +------------'--+  Information +-'---------------+  |
  |               '                   '                  |
  |               ' & Identity        ' Identity         |
  |   Attestation ' Information       ' Information      |
  |               v                   v                  |
  |------------------------------------------------------|
  | Host Operating System and Hardware                   |
  +------------------------------------------------------+
~~~~
{: #arch-fig title="Host Software Layinger in a Workload Identity Architecture."}

Once the workload is started and has obtained unique identity information, it can offer its services. Once a service is invoked on a workload it may require interaction with other workloads. An example of such interaction is shown in {{?I-D.ietf-oauth-transaction-tokens}} where an externally-facing endpoint is invoked using conventional authorization mechanism, such as an OAuth 2.0 access token. The interaction with other workload may require the security context to be passed along the call chain.

In the rest of the document we describe terminology and use cases, discuss details of the architecture, and discuss threats.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the following terms:

* Workload

A workload is a running instance of software executing for a specific purpose that interacts with other parts of a larger system. A workload may exist for a very short durations of time (nanoseconds) and run for a specific purpose such as to provide a response to an API request. Other kinds of workloads may execute for a very long duration such as months or years - examples include database services and machine learning training jobs.

* Security Context

A security context contains information needed for a workload to pefrom its function. This information is often used for authorization, accounting and auditing purposes and often contains information about the request being made. Some examples inlcude user information, software and hardware information or information about what processing has already happened for the request. Different pieces of context information may originate from different authorities.

* Identity Proxy

Identity proxy is an intermediary that can inspect, replace or augment workload identity and security context information. Identity proxy can be a capability of a transparent network service, such as a security gateway, or it can be implemented in service performing explicit connection processing, such as a reverse proxy or a CDN service.

# Architecture

## Initial Workload Identity

Typically a workload obtains its identity early in its lifecycle. This identity is sometimes referred to as the "bottom turtle" on which further identity is built. Some common mechanisms for obtaining this initial identity include:

* File System projection - in this mechanisms the identity is provisioned to the workload as an entity in the filesystem.
* Local API - the identity is provided through an api such as a local domain socket (SPIFFE) or local network API calls (Cloud Provider Metadata Server).
* Environment Variables - identity may also be injected into workloads using operating system environment variables.

## Attestation

## Identity Credentials

The identity is provisioned to the workload as a set of credentials. There are two main types of workload credentials: bearer tokens and X.509 certificates.

Bearer tokens are tokens presented to another party as proof of identity.  They are typically signed to prevent forgery, however since these credentials are not bound to other information its possible that they could be stolen and reused elsewhere. To reduce some of these risks, bearer tokens may have short lifespans and may be rotated often.

X.509 certificate credentials consist of two parts, a public key certificate that is a signed data structure that contains a public key and identity information and a private key which. The certificate is sent during authentication, however the private key is kept secret and only used in cryptographic computation to to prove that the presenter has access to the private that corresponds to the public key in the certificate.

## Basic Service Authentication

One of the most basic use cases for workload identity is for authenticating one workload to another such as in the case where one service is making a request of another service within a larger application. Even in this simple case the identity of the workload is often a composite of many attributes such as:

* Service Name
* Instance Name
* Role
* Environment
* Trust Domain
* Software Attestation
* Hardware Attestation

These attributes are used for various purposes:

* ensuring the request is made to the correct service or service instance
* authorizing access to APIs and resources
* providing an audit train of requests within a system
* providing context for other decisions made within a service

There are several methods defined to perform this authentication.  Some of the most common include:

* TLS authentication of the server using X.509 certificates and bearer token, encoded as JWTs, on top of HTTP or other protocol request.
* Mutual TLS authentication using X.509 certificate for both client and server.
* TLS authentication of the server and HTTP request signing using a secret key.

## Security Context Establishment and Propagation

In a typical system of workloads additional information is needed in order for the workload to perform its function. For example, it is common for a workload to require information about a user or other entity that originated the request. Other types of information may include information about the hardware or software that the workload is running or information about what processing and validation has already been done to the request.  This type of information is part of the security context that the workload uses during authorization, accounting and auditing.  This context is propagated and possibly augmented from workload to workload using tokens. Workload identity comes into play to ensure that the information in the context can only be used by an authorized workload and that the context information originated from an authorized workload.

## Delegation and Impersonation

TBD.

## Asynchronous and Batch Requests

TBD.

## Cross-boundary Workload Identity

As workloads often need to communicate across administrative boundaries, extra care needs to be taken when it comes to identity communication to ensure scalability and privacy.

### Egress Identity Generalization

A workload communicating with a service, or another workload provided by external organization may need to provide more generic identity information. Detailed identity of internal workload originating the communication is relevant inside the administrative domain but could be excessive for the outside world and expose internal topology information that can be sensitive.

A security gateway at the edge of administrative domain can be used to validate identity information of the workload, perform context specific authorization of the transaction and replace workload specific identity with a generalized one for given administrative domain.

### Inbound Gateway Identity Validation

Inbound security gateway is a common design pattern for service protection. This functionality is often found in CDN services, API gateways, load balancers, Web Application Firewalls (WAFs) and other security solutions. Workload identity verification should be performed as a part of these security services. After validation of workload identity, the gateway may either leave it unmodified or replace it with its own identity to be validated by the destination.

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
