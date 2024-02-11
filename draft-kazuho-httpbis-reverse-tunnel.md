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

This document specifies an HTTP extension to establish bi-directional byte
streams in the direction from servers to their clients, utilizing HTTP as a
tunneling mechanism. This approach facilitates the operation of servers located
behind firewalls, which accept connections through TCP relays, as well as
application-protocol-specific servers, such as HTTP origins connecting to HTTP
proxies.


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
Host: example.com
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
acting as the TLS {{?TLS=RFC8446}} server) SHOULD use one of the TLS-based
authentication schemes to identify itself.

The server SHOULD authenticate itself either by using one of the HTTP-based
authentication schemes (e.g., HTTP Basic Authentication {{?BASIC-AUTH=RFC7617}})
or by using one of the TLS-based authentication schemes when HTTPS is used.


# Relaying Connections

When the client acts as a transport-layer protocol relay (i.e., forwarding TCP
connections), it is crucial for the reverse tunnel protocol to both signal the
establishment of the relayed connection and identify the client-side of the
connection being relayed.

Upon receiving a request to establish a reverse tunnel, a client acting as a
relay SHOULD initially send an informational HTTP response with the status code
100 (Continue). This indicates that the client may serve as a tunnel, but no
connections are immediately available to be relayed.

Once a connection eligible for relay becomes available, the client sends a
successful response (i.e., 101 Switching Protocols or 200 OK, depending on the
HTTP protocol version in use) to indicate the commencement of its relay
operations.

This successful response SHOULD include the "Forwarded" header field
{{!FORWARDED=RFC7239}} that identifies the client side of the connection being
relayed.

The client MAY issue 100 (Continue) responses multiple times.

If the client instantly matches a connection to be relayed upon receiving a
tunnel establishment request, the client MAY omit the 100 (Continue) response
and directly send a successful response.

{{fig-tcp-relay}} illustrates an exchange of HTTP/1.1 messages to establish a
reverse TCP relay.

~~~
GET /reverse-tcp-relay HTTP/1.1
Host: example.com
Connection: upgrade
Upgrade: reverse

HTTP/1.1 100 Continue

HTTP/1.1 101 Switching Protocols
Connection: upgrade
Upgrade: reverse
Forwarded: for=192.0.2.43

~~~
{: #fig-tcp-relay title="Establishing a TCP relay"}

When the client is forwarding at the application protocol layer, rather than
relaying the bytes exchanged on the transport, the client MAY use mechanisms
provided by the application protocols to convey the identity of the client being
relayed. For instance, a client acting as an HTTP proxy can forward HTTP
requests to servers on the HTTP request level, incorporating the IP address of
the original HTTP clients by adding the "Forwarded" header field to each HTTP
request it forwards.


# Application-Layer Protocol Negotiation

While TLS {{TLS}} can be used on top of an established tunnel, doing so might
not be necessary for ensuring the security of communication if the tunnel is
established via HTTPS, and the client side of the reverse tunnel also functions
as the client side of the application protocol in use. A typical scenario
involves an HTTPS reverse proxy serving as the client of a reverse tunnel. This
proxy terminates incoming TLS connections and decrypts the HTTP requests before
forwarding them through the reverse tunnel, which is secured by a separate TLS
connection.

In these deployments, foregoing the use of TLS above the established tunnel can
yield performance benefits without compromising security. However, this approach
requires that endpoints negotiate the application protocol without relying on
the Application-Layer Protocol Negotiation {{!ALPN=RFC7301}} performed during
the TLS handshake.

To address this need, this document introduces an HTTP header-based mechanism
for negotiating the application protocol. It employs ALPN identifiers for naming
the application protocols, allowing for the selection of existing application
protocols without depending on TLS-based negotiation.


## Indicating Protocols Available for Use

To indicate the application protocols that the server is willing to utilize, a
server MAY include an "ALPN" header field {{!ALPN-HEADER=RFC7639}} in the HTTP
request that it issues. The "ALPN" header field carries a list of
application-protocol identifies that the server is willing to use.

{{fig-alpn-request}} shows an HTTP/1.1 request attempting to establish a reverse
channel that uses either HTTP/2 {{?HTTP2=RFC9113}} or HTTP/1.1
{{?HTTP1=RFC9112}} as the application protocol.

~~~
GET /reverse-endpoint HTTP/1.1
Host: example.com
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

When a server sends an HTTP request with an "ALPN" header field but receives a
successful response without a "Selected-ALPN" header field, it could either be
an indication that the client chose an application protocol that the server did
not offer, or that the server could not determine which application protocol has
been chosen. Therefore, the client SHOULD NOT assume that an application
protocol other than the ones being offered has been selected.


# IANA Considerations

TBD.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
