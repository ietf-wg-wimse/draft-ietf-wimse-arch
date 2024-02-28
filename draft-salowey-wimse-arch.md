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
    organization: Siemens
    email: hannes.tschofenig@gmx.net



normative:

informative:


--- abstract

TODO Abstract


--- middle

# Introduction

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}

## Definitions

* Workload

A workload is a running instance of software executing for a specific purpose that interacts with other parts of a larger system. A workload may exist for a very short durations of time (nanoseconds) and run for a specific purpose such as to provide a response to an API request. Other kinds of workloads may execute for a very long duration such as months or years - examples include database services and machine learning training jobs.


# Use Cases

1. Basic Service Authentication

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

* TLS authentication of the server using X.509 certificates and bearer token JWTs within an HTTP or other protocol request.
* Mutual TLS authentication using X.509 certificate for both client and server
* TLS authentication of the server and HTTP request signing using a secret key

2. Additional Context Establishment

3. Asynchronous Requests

4. Scheduled Batch Requests



# Architecture




# Security Considerations

## Threat Model

### Traffic Interception

Workloads communicating within an an applications may face different threats to traffic interception in different deployments. In many deployments security controls are deployed for internal communications at lower layers to reduce the risk of traffic observation and modification for network communications. When a security layer such as TLS is deployed in these environments TLS may be termiated in various places including the workload itself and in various middleware devices such as load balancers, gateways, proxies, and firewalls. Therefore protection is provided only between each adjacent pair of TLS endpoints. There are no guarantees of confidentiality, integrity and correct identity passthrough in those middleware devices and services.

### Information Disclosure

Observation and interception of network traffic is not the only means of disclosure in these systems. Other vectors of information leakage is through disclosure in log files and other observability and troubleshooting mechanisms. For example, an application may log the contents of HTTP headers containing JWT bearer tokens.  The information in this logs may be made available to other systems with less stringent access controls which may result in this token falling into an attackers hands who then uses it to compromise a system. This creates privacy risks and potential surface for reconnaissance attacks. If observed tokens can be reused, this also may allow attackers to impersonate workloads.

### Workload Compromise

Even the most well-designed and implemented workloads may contain security flaws that allow an attacker to gain limited or full compromise. For example, a server side request forgery may result in the ability for an attacker to force the workload to make requests of other parts of a system even though the rest of the workload functionality may be unaffected. An attacker with this advantage may be able to utilize privileges of the compromised workload to attack other parts of the system. Therefore it is important that communicating workloads apply the principle of least privilege through security controls such as authorization.

## Mitigating Token Theft




# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
