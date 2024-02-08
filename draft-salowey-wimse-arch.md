---
title: "Workload Identity in a Multi System Environment (WIMSE) Architecture"
abbrev: "WIMSE Architecture"
docname: draft-salowey-wimse-arch-latest
category: info

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
    organization:
    email:
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

A workload is a process that performs one or mote functions within a system. A workload will often make requests to and or service requests from other workloads and components within a system. Some workloads may be replicas of each other in order to scale a particular function within the system. A workload is often deployed in a contain virtual machine or physical host, but a collection of these may be considered a workload if it is identified to the rest of the system as a single entity.

# Use Cases

# Architecture



# Security Considerations

# Threat Model
TODO Security


# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
