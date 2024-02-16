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
behind firewalls that communicate with known clients. The tunnel can also be
used by TCP relays to forward incoming TCP connections to servers behind
firewalls.


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

This document proposes an alternative to utilizing VPNs by defining a method for
servers to establish connections to clients over HTTP, thereby creating
bi-directional byte streams for application protocols. Specifically, this
extension employs HTTP upgrades in HTTP/1.1 ({{HTTP-SEMANTICS}} Section 7.8) and
the "extended CONNECT" method in HTTP/2 {{!EXTENDED-CONNECT-H2=RFC8441}} and
HTTP/3 {{!EXTENDED-CONNECT-H3=RFC9220}} for the establishment of these byte
streams.

Beyond better manageability and reduced performance overhead in comparison to
VPNs, this method of employing HTTP for tunnel establishment provides additional
advantages. It enables endpoints to utilize a variety of authentication schemes
that are native to HTTP, such as HTTP authentication and cookies. Furthermore,
it shifts client identification from relying on IP addresses and port numbers to
using URIs, thereby enhancing flexibility and integration with web technologies.

As servers specify their clients using URIs, only clients known to a server can
communicate directly with it. Nevertheless, clients have the capability to
forward application protocol-level messages (e.g., HTTP requests) or relay TCP
connections they receive or accept from the Internet. Through such relay
mechanisms, servers hidden behind firewalls can effectively receive requests
from any client on the Internet, bridging the gap between restricted access and
global connectivity.


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

In HTTP/1.1, the HTTP upgrade mechanism ({{HTTP-SEMANTICS}} Section 7.8) is
used.

The method of the issued request SHALL be "GET".

The upgrade token is conveyed by the "Upgrade" header field, and once the
reverse tunnel is established successfully, the client responds with a 101
(Swithing Protocols) response.

{{fig-tunnel-establishment}} shows an exchange of HTTP/1.1 request and response
establishing a reverse tunnel. In this example, the Basic HTTP Authentication
Scheme {{?BASIC-AUTH=RFC7617}} is used to authenticate the server.

~~~
GET /reverse-endpoint HTTP/1.1
Host: example.com
Connection: upgrade
Upgrade: reverse
Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==

HTTP/1.1 101 Switching Protocols
Connection: upgrade
Upgrade: reverse

~~~
{: #fig-tunnel-establishment title="Establishing Reverse Tunnel over HTTP/1.1"}

## HTTP/2 and HTTP/3

In HTTP/2 and HTTP/3, extended CONNECT is used; see {{EXTENDED-CONNECT-H2}} and
{{EXTENDED-CONNECT-H3}}.

In both HTTP versions, the method being used is "CONNECT" and the upgrade
token is conveyed by the ":protocol" pseudo header. Once the reverse tunnel is
established successfully, the client responds with a 200 (OK) response.


# Authentication

When HTTPS is used for establishing the tunnel, clients (i.e., the nodes acting
as TLS {{?TLS=RFC8446}} servers) SHOULD use one of the TLS-based authentication
schemes to identify themselves.

Servers SHOULD authenticate themselves either by using one of the HTTP-based
authentication schemes; e.g., HTTP Authentication ({{HTTP-SEMANTICS}}
Section 11), or, when HTTPS is used, by using one of the TLS-based
authentication schemes.


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

{{fig-alpn-request}} shows an exchange of HTTP/1.1 response and response that
sets up a tunnel for forwarding HTTP requests using HTTP/2, where tunnel server
is the origin and the tunnel client is the reverse proxy.

~~~
GET /reverse-connect/proxy/for/service/X HTTP/1.1
Host: example.com
Connection: upgrade
Upgrade: reverse
Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==
ALPN: h2, http%2F1.1

HTTP/1.1 101 Switching Protocols
Connection: upgrade
Upgrade: reverse
Selected-ALPN: h2

~~~
{: #fig-alpn-request title="Setting up a HTTP/2 Tunnel for forwarding HTTP Requests"}

When a server sends an HTTP request with an "ALPN" header field but receives a
successful response without a "Selected-ALPN" header field, it could either be
an indication that the client chose an application protocol that the server did
not offer, or that the server could not determine which application protocol has
been chosen. Therefore, the client SHOULD NOT assume that an application
protocol other than the ones being offered has been selected.


# Relaying Connections

When a client is forwarding at the application protocwl layer, it can utilize
mechanisms provided by the application protocol in use to convey the identity of
the actual client from which messages were received. For example, HTTP
intermediaries acting as reverse tunnel clients can add the "Fowarded" header
field {{!FORWARDED=RFC7239}} to each request they relay.

However, when the client acts as a transport-layer protocol relay (i.e.,
relaying TCP connections), it becomes the responsibility of the reverse tunnel
protocol to convey the client-side idenity of the TCP connection being relayed.

Upon receiving a request to establish a reverse tunnel, a client acting as a
relay SHOULD match a connection to be relayed before sending a successful
response (i.e., 101 Switching Protocols or 200 OK, depending on the HTTP
protocol version in use). Once a connection has been matched, the client SHOULD
send a successful response with a "Forwarded" header field {{FORWARDED}}
carrying the client-side identity of the TCP connection being relayed. After
that, the client begins relaying the bytes being sent and received between the
tunnel and the matched connection.

If the client cannot immediately match a connection to be relayed, the client
MAY send an informational response of 100 (Continue) to acknowledge that it has
received the request but it is not yet ready to send a final response.

This informational response MAY be sent more than once.

When the client gives up waiting for a matching connection to become available,
the client SHOULD send a 204 (No Content) response to indicate that it
successfully processed the request, but a matching connection was not
available.

{{fig-tcp-relay}} illustrates an exchange of HTTP/1.1 messages to establish a
reverse TCP relay.

~~~
GET /.well-known/listen-tcp/0.0.0.0/25/ HTTP/1.1
Host: example.com
Connection: upgrade
Upgrade: reverse
Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==

HTTP/1.1 100 Continue

HTTP/1.1 101 Switching Protocols
Connection: upgrade
Upgrade: reverse
Forwarded: for=192.0.2.43

~~~
{: #fig-tcp-relay title="Establishing a TCP relay for SMTP"}


## Specifying the Listening Address and Port

Clients acting as relays that allow servers specify the listening address or
port SHOULD use the following URI Template {{!TEMPLATE=RFC6570}} to define the
target URI on which they provide the service. Adopting this template simplifies
operations by ensuring a uniform method for configuring endpoints. Examples are
shown below:

~~~
https://example.com/.well-known/reverse/tcp/{listen_host}/{listen_port}/
https://example.org/listen/to/tcp?h={listen_host}&p={listen_port}
~~~

Furthermore, the use of the default template is RECOMMENDED, which is defined as
"https://$CLIENT_HOST:$CLIENT_PORT/.well-known/reverse/tcp/{listen_host}/{listen_port}/",
where $CLIENT_HOST and $CLIENT_PORT are the host and port of the client.

The "listen_host" variable specifies the listening address. This variable MAY
contain an wildcard address.

The "listen_port" variable specifies the listening port.

The following requirements apply to the URI Template:

* The URI Template MUST be a level 3 template or lower.

* The URI Template MUST be in absolute form and MUST include non-empty scheme,
  authority, and path components.

* The path component of the URI Template MUST start with a slash ("/").

* All template variables MUST be within the path or query components of the URI.

* The URI template MUST contain the two variables "listen_host" and
  "listen_port", and MAY contain other variables.

* The URI Template MUST NOT contain any non-ASCII Unicode characters and MUST
  only contain ASCII characters in the range 0x21-0x7E inclusive (note that
  percent-encoding is allowed; see {{!URI=RFC3886}} Section 2.1).

*  The URI Template MUST NOT use Reserved Expansion ("+" operator), Fragment
   Expansion ("#" operator), Label Expansion with Dot-Prefix, Path Segment
   Expansion with Slash-Prefix, nor Path-Style Parameter Expansion with
   Semicolon-Prefix.

Servers SHOULD validate the requirements above; however, servers MAY use a
general-purpose URI Template implementation that lacks this specific validation.
If a server detects that any of the requirements above are not met by a URI
Template, the server MUST reject its configuration and abort the request without
sending it to the relaying client.


# IANA Considerations

Once approved, this document will request IANA to register the following entries
to the respective registries.

To the "HTTP Upgrade Tokens" registry maintained at
<https://www.iana.org/assignments/http-upgrade-tokens>:

Value:
: reverse

Description:
: Reverse Tunnel

Expected Version Tokens:
: None

Reference:
: this document

To the "Hypertext Transfer Protocol (HTTP) Field Name Registry" maintained at
<https://www.iana.org/assignments/http-fields>:

Field Name:
: Selected-ALPN

Template:
: None

Status:
: permanent

Reference:
: this document

Comments:
: None

To the "Well-Known URIs" registry maintained at
<https://www.iana.org/assignments/well-known-uris>:

URI Suffix:
: listen-tcp

Change Controller:
: IETF

Reference:
: this document

Status:
: permanent

Related Information:
: Includes all resources identified with the path prefix
"./well-known/listen-tcp/".


--- back

# Acknowledgments
{:numbered="false"}

This document is inspired by {{?I-D.bt-httpbis-reverse-http-01}} and the
discussion at IETF 118.
