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

As servers specify their clients using URIs, only clients known to a server can
communicate directly with it. However, clients can act as relays, forwarding TCP
connections from the Internet to the servers connected through the reverse
tunnel.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# The Protocol

To setup the bi-directional byte streams, servers connect to the clients as
specified by the URI, and issues an HTTP request to establish a tunnel.

To signal the intent to establish a reverse tunnel, an upgrade token named
"reverse" is used.

The method and the conveyor of the upgrade token are different between the HTTP
versions.


## HTTP/1.1

In HTTP/1.1, method of the issued request SHALL be "GET" and the upgrade token
SHALL be conveyed by the "Upgrade" header field.

As an upgrade is initiated, the "Connection" header specifies the "Upgrade"
option ({{HTTP-SEMANTICS}} Section 7.8).

Once the reverse tunnel is established successfuly, the client responds with a
101 (Swithing Protocols) response.


## HTTP/2 and HTTP/3

In HTTP/2 and HTTP/3, extended CONNECT is used ({{EXTENDED-CONNECT-H2}} or
{{EXTENDED-CONNECT-H3}}).

In either version of HTTP, the method being used is "CONNECT" and the upgrade
token is conveyed by the ":protocol" pseudo header.

Once the reverse tunnel is established successfully, the client responds with a
200 (OK) response.


# Authentication

When HTTPS is used for establishing the tunnel, the client (i.e., the node
acting as the TLS server) SHOULD use one of the TLS-based authentication schemes
to identify itself.

The server SHOULD authenticate itself either by using one of the HTTP-based
authentication schemes (e.g., HTTP Basic Authentication {{?BASIC-AUTH=RFC7617}})
or by using one of the TLS-based authentication schemes when HTTPS is used.


# Forwarding Client Address

When the client is acting as a TCP relay, it MAY include the "Forwarded" header
field {{!FORWARDED=RFC7239}} in the HTTP response it sends, to indicate the
client identity of the relayed connection.


# Protocol Negotiation

TBD


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
