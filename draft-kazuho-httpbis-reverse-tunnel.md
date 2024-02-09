---
title: "Reverse Tunnel over HTTP"
docname: draft-kazuho-httpbis-reverse-tunnel-latest
category: std
wg: httpbis
ipr: trust200902
keyword: internet-draft
stand_alone: yes
pi: [toc, sortrefs, symrefs]
author:
 -
    fullname: Kazuho Oku
    organization: Fastly
    email: kazuhooku@gmail.com
normative:
  HTTP-SEMANTICS: RFC9110

--- abstract

This document specifies the protocol for setting up bi-directional byte streams
from application servers to their clients by using HTTP as a tunnel.


--- middle

# Introduction

In typical application protocols operating on top of TCP, clients initiate TCP
connections by specifying the server's IP address and the port number. However,
not all servers can accept incoming TCP connections directly.

Presently, servers situated behind firewalls that block incoming TCP connections
often rely on virtual private networks (VPNs). These VPNs enable the passage of
packets initiating TCP connections to servers through encapsulation, effectively
bypassing firewall restrictions. This approach, however, compromises network
manageability and incurs performance penalties due to the additional routing and
encapsulation involved.

This document proposes an alternative method.

Instead of utilizing VPNs, it describes how servers establish connections to
clients over HTTP to create bi-directional byte streams for applications to
exchange information. Specifically, this protocol leverages HTTP upgrades in
HTTP/1.1 ({{HTTP-SEMANTICS}} Section 7.8) or the "extended CONNECT" method of
HTTP/2 ({{!EXTENDED-CONNECT-H2=RFC8441}}) and HTTP/3
({{!EXTENDED-CONNECT-H3=RFC9220}}) to establish connections.

Employing HTTP for connection establishment offers additional benefits. Beyond
TLS-based authentication schemes, servers can authenticate themselves using
varios HTTP-provided authentication schemes, such as HTTP authentication and
cookies. Furthermore, clients are identified by URI rather than by IP address
and port number, enhancing flexibility and integration with web technologies.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# The Protocol

TBD


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
