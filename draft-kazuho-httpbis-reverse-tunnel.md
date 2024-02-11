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

This document specifies the HTTP extension for setting up bi-directional byte
streams from application servers to their clients by using HTTP as a tunnel.


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
connections or application-level messages (e.g., HTTP requests) that they accept
or receive from the Internet to the servers connected through the reverse
tunnel.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# The Protocol

To setup a reverse tunnel, a server connects to the client as specified by the
URI and issues an HTTP request.

To signal the intent to establish a reverse tunnel, an upgrade token named
"reverse" is used.

The method and the conveyor of the upgrade token are different between the HTTP
versions.


## HTTP/1.1

In HTTP/1.1, the HTTP upgrade mechanism is used ({{HTTP-SEMANTICS}} Section
7.8).

The method of the issued request SHALL be "GET".

The upgrade token is conveyed by the "Upgrade" header field, and once the
reverse tunnel is established successfuly, the client responds with a 101
(Swithing Protocols) response.

{{fig-tunnel-establishment}} shows an exchange of HTTP/1.1 request and response
establishing a reverse tunnel.

~~~
GET /reverse-endpoint HTTP/1.1
Connection: upgrade
Upgrade: reverse

HTTP/1.1 101 Switching Protocols
Connection: upgrade
Upgrade: reverse

~~~
{: #fig-tunnel-establishment title="Establishing Reverse Tunnel over HTTP/1.1"}

## HTTP/2 and HTTP/3

In HTTP/2 and HTTP/3, extended CONNECT is used ({{EXTENDED-CONNECT-H2}} or
{{EXTENDED-CONNECT-H3}}).

In both HTTP versions, the method being used is "CONNECT" and the upgrade
token is conveyed by the ":protocol" pseudo header. Once the reverse tunnel is
established successfully, the client responds with a 200 (OK) response.


# Authentication

When HTTPS is used for establishing the tunnel, the client (i.e., the node
acting as the TLS server) SHOULD use one of the TLS-based authentication schemes
to identify itself.

The server SHOULD authenticate itself either by using one of the HTTP-based
authentication schemes (e.g., HTTP Basic Authentication {{?BASIC-AUTH=RFC7617}})
or by using one of the TLS-based authentication schemes when HTTPS is used.


# Forwarding Client Address

When sending a successful HTTP response, clients MAY include the "Forwarded"
header field {{!FORWARDED=RFC7239}} in the HTTP response to indicate the client
identity of the relayed connection. For clients acting as a transport-layer
relay, use of the "Forwarded" response header field is the only way to relay the
identity.

{{fig-client-address}} shows an HTTP/1.1 response conveying the client IP
address of the connection being relayed.

~~~
HTTP/1.1 101 Switching Protocols
Connection: upgrade
Upgrade: reverse
Forwarded: for=192.0.2.43

~~~
{: #fig-client-address title="Response with a Forwarded Header Field"}

When the client is an application-protocol relay, it MAY use the mechanism
provided by the application protocols to relay the identity of the client being
relayed. For example, a client acting as a HTTP proxy can forward HTTP requests
to servers at the HTTP request level, indicating the IP address of the HTTP
clients by adding the Forwarded header field to each of the HTTP request that it
forwards.


# Application-Layer Protocol Negotiation

As the use of TLS {{?TLS=RFC8446}} becomes prevalent, an increasing number of
application protocols depend on the Application-Layer Protocol Negotiation
Extension {{!ALPN=RFC7301}} to determine the application protocol to utilize.

While TLS can be used on top of the established tunnel and the application
protocol can be negotiated during the TLS handshake, doing so is not necessary
to achieve the goal of encryption when the tunnel is established via HTTPS. When
operating over HTTPS, some deployments could opt to exchange messages without
having another layer of encryption above the tunnel.

This document specifies an HTTP header-based mechanism for negotiating the
application protocol. ALPN identifiers are used for naming the application
protocols, so that existing application procotols can be selected.


## Indicating Protocols Available for Use

To indicate the application protocols that the server is willing to utilize, a
server MAY include an "ALPN" header field {{!ALPN-HEADER=RFC7639}} in the HTTP
request that it issues. The "ALPN" header field carries a list of
application-protocol identifies that the server is willing to use.

{{fig-alpn-request}} shows an HTTP/1.1 request attempting to establish a reverse
channel that uses either HTTP/2 {{?HTTP2=RFC9113}} or HTTP/1.1
{{?HTTP1=RFC9112}} as the application protocol.

~~~
GET /endpoint HTTP/1.1
Connection: upgrade
Upgrade: reverse
ALPN: h2, http%2F1.1

~~~
{: #fig-alpn-request title="Request with an ALPN Header Field"}


## Indicating the Chosen Protocol

When a client receives an HTTP request with an "ALPN" header field, and if the
client decides to select one of the application protocols being offered, the
client includes a "Selected-ALPN" header field in the HTTP response.

If the HTTP request does not include an "ALPN" header field, the client MUST NOT
send a "Selected-ALPN" header field.

Syntax of the "Selected-ALPN" header field is as follows. The syntax reuses the
encoding of the "ALPN" header field, but always includes exactly one
application-protocol identifier that is being chosen.

~~~
Selected-ALPN = ALPN
~~~

{{fig-alpn-response}} shows an HTTP/1.1 response indicating that the tunnel has
been established with the chosen protocol being HTTP/2.

~~~
HTTP/1.1 101 Switching Protocols
Connection: upgrade
Upgrade: reverse
Selected-ALPN: h2

~~~
{: #fig-alpn-response title="Response with a Selected-ALPN Header Field"}

When a server receives a successful HTTP response that does not carry the
"Selected-ALPN" header field, it could either be an indication that the server
chose another application protocol or that the server could not determine which
application protocol has been chosen. Therefore, the client SHOULD NOT assume
that an application protocol other than the ones being offered has been
selected.


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
