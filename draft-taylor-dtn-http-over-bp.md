---

title: "HTTP over the Bundle Protocol"
abbrev: "HTTP over BP"
category: std
ipr: trust200902

docname: draft-taylor-dtn-http-over-bp-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Internet"
workgroup: "Delay/Disruption Tolerant Networking"
keyword:

- DTN
- BPv7
- Bundle Protocol
- HTTP
- Binary HTTP
- Caching
venue:
  group: "Delay/Disruption Tolerant Networking"
  type: "Working Group"
  mail: "dtn@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/dtn/"
  github: "ricktaylor/http-over-bp"
  latest: "https://ricktaylor.github.io/http-over-bp/draft-taylor-dtn-http-over-bp.html"

author:

- fullname: Rick Taylor
  organization: Aalyria Technologies
  email: rtaylor@aalyria.com

normative:
  RFC8610:
  RFC8949:
  RFC9110:
  RFC9111:
  RFC9171:
  RFC9172:
  RFC9173:
  RFC9209:
  RFC9292:
  RFC9758:

informative:
  RFC3135:
  RFC4838:
  RFC5861:
  RFC6455:
  RFC6585:
  RFC7240:
  RFC8075:
  RFC8126:
  RFC8188:
  RFC8216:
  RFC8441:
  RFC8586:
  RFC9112:
  RFC9113:
  RFC9114:
  RFC9213:
  RFC9220:
  RFC9421:
  RFC9440:
  RFC9457:
  RFC9460:
  RFC9499:
  RFC9530:
  I-D.blanchet-dtn-http-over-bp:
  I-D.ietf-core-coap-bp:
  I-D.ietf-httpapi-idempotency-key-header:
  I-D.ietf-httpapi-ratelimit-headers:
  I-D.ietf-httpbis-resumable-upload:
  HTML:
    title: "HTML Living Standard: Server-sent events"
    author:
      - org: WHATWG
    date: false
    target: https://html.spec.whatwg.org/multipage/server-sent-events.html
  FETCH-METADATA:
    title: "Fetch Metadata Request Headers"
    author:
      - org: W3C
    date: false
    target: https://www.w3.org/TR/fetch-metadata/

--- abstract

This document specifies a mechanism for carrying HTTP over Bundle Protocol Version 7 (BPv7) networks. Each HTTP request and each HTTP response travels as a single bundle, encoded as a Binary HTTP message inside a small CBOR envelope that correlates responses with requests. Because every bundle carries a complete, self-contained message, the mechanism needs none of the reliable, ordered, connection-oriented transport services that HTTP otherwise relies on, and it tolerates the delay, disruption, reordering, and duplication of a Delay-Tolerant Network. HTTP semantics are unchanged; only the transport beneath them is replaced. The mechanism enables HTTP application-layer gateways at the edges of a bundle network, through which unmodified HTTP clients and origin servers communicate across it, and also lets BP-native applications act as HTTP clients or origin servers directly. The document describes how gateways use HTTP caching to remain useful when round trips are long or intermittent, and it requests a well-known ipn service number.

--- middle

# Introduction

This document specifies a mechanism for carrying HTTP over networks that use Bundle Protocol Version 7 (BPv7) {{RFC9171}}: HTTP over BP. HTTP semantics {{RFC9110}} are unchanged; only the transport beneath them is replaced. Each HTTP request and each HTTP response is encoded as a Binary HTTP message {{RFC9292}} and carried as the application data unit (ADU) of a single bundle, inside a small CBOR {{RFC8949}} envelope that correlates each response with its request ({{envelope}}). No connection, stream, or sequence state spans the bundle network. Every bundle can be interpreted on its own, so reordering, duplication, and arbitrary delay need no recovery mechanism, and the loss of a bundle loses one exchange, never a connection.

Any BP node can use HTTP over BP to send HTTP requests or to answer them, whether or not that node also speaks HTTP over IP. {{applicability}} describes the uses of the mechanism and how it compares with the alternatives.

## Scope {#scope}

This document specifies:

- the format of the ADUs that carry HTTP messages ({{format}});
- the procedures by which a requester and a responder exchange them ({{procedures}}), including responses that are produced incrementally and event streams ({{extended}}); and
- requirements and guidance for gateways that preserve HTTP semantics across a bundle network, chiefly caching ({{gateways}}).

This document builds on existing specifications wherever it can. The following functions are outside its scope; most are already covered by existing HTTP, DNS, or DTN specifications, which apply to gateways unchanged.

Directing clients to a gateway:
: Clients reach a client-side gateway by the usual means of reaching an HTTP intermediary: by configuring it as a proxy, so that requests carry absolute-form targets ({{Section 3.2.2 of RFC9112}}); or through DNS, by resolving the names of origins to the addresses of the gateway, for example with split-horizon DNS ({{Section 6 of RFC9499}}).

Selecting a responder:
: The mapping from an authority to the EID of a responder can be configured, or obtained from a resolution service, for example one built on DNS Service Binding records {{RFC9460}}. {{discovery}} gives the requirements that any such mechanism has to meet.

> A companion document is expected to specify an SVCB-based mechanism.
{:cref}

Admission and capacity policy:
: Which requests a gateway admits, and how it rations the capacity of a bundle path, are matters for each deployment. Gateways convey their decisions with existing HTTP mechanisms: 413 (Content Too Large), and 503 (Service Unavailable) with Retry-After {{RFC9110}}; 429 (Too Many Requests) {{RFC6585}}; and Problem Details {{RFC9457}}. Quota policies can be advertised with the RateLimit fields {{I-D.ietf-httpapi-ratelimit-headers}}.

Asynchronous and repeatable requests:
: Clients can ask for asynchronous processing with `Prefer: respond-async` {{RFC7240}}, resubmit unsafe requests without repeating their effect using the Idempotency-Key field {{I-D.ietf-httpapi-idempotency-key-header}}, and divide large uploads into independent steps with Resumable Uploads for HTTP {{I-D.ietf-httpbis-resumable-upload}}. {{client-facing}} describes how these apply at a gateway.

End-to-end security:
: TLS terminates at the gateways ({{security}}). Mechanisms that protect HTTP messages, rather than connections, pass through gateways unchanged: HTTP Message Signatures {{RFC9421}} authenticate messages, Encrypted Content-Encoding {{RFC8188}} protects content, and Digest Fields {{RFC9530}} protect its integrity.

Cache control for DTN gateways:
: Origin servers control caching at gateways with ordinary Cache-Control directives {{RFC9111}}. Targeted Cache-Control fields {{RFC9213}} provide the means by which directives could be addressed to DTN gateways alone; defining such a field is left to a future document.

Streaming media:
: Segmented media formats such as HTTP Live Streaming {{RFC8216}} were designed to traverse HTTP caches, and cross a gateway pair without adaptation.

Other application protocols:
: CoAP, which already runs over a message-oriented transport, is carried over BP by {{I-D.ietf-core-coap-bp}}; existing mappings between HTTP and CoAP {{RFC8075}} apply between the two. Tunnelling (CONNECT), WebSocket, and other protocols that require an interactive end-to-end connection are not supported ({{unsupported}}).

# Conventions and Terminology

{::boilerplate bcp14-tagged}

This document uses the terms "bundle", "application data unit (ADU)", "bundle protocol agent (BPA)", "endpoint ID (EID)", "node ID", "lifetime", "creation timestamp", and "bundle age" from {{RFC9171}}; the terms "intermediary", "gateway", "origin server", "user agent", "target URI", "safe", and "idempotent" from {{RFC9110}}; and the terms "shared cache", "stored response", "fresh", and "stale" from {{RFC9111}}. Data structures are described using the Concise Data Definition Language (CDDL) {{RFC8610}}.

Requester:
: A BP node that sends HTTP requests over BP as specified in this document and receives the corresponding responses. A requester is either a client-side gateway or a BP-native HTTP client.

Responder:
: A BP node that receives HTTP requests over BP as specified in this document and returns responses. A responder is either an origin-side gateway or a BP-native HTTP origin server.

Client-side gateway:
: A requester that receives HTTP requests from clients over conventional HTTP connections and forwards them over BP.

Origin-side gateway:
: A responder that forwards the HTTP requests it receives over BP to origin servers over conventional HTTP connections, and returns their responses over BP.

Request ADU:
: An ADU, formatted as specified in {{format}}, that carries an HTTP request.

Response ADU:
: An ADU, formatted as specified in {{format}}, that carries an HTTP response.

Exchange:
: A request ADU together with the response ADU or ADUs that answer it, identified by a correlation identifier ({{correlation}}).

# Applicability {#applicability}

HTTP over BP has two principal uses: HTTP application-layer gateways ({{use-gateways}}) and BP-native HTTP applications ({{use-native}}). {{alternatives}} compares it with other ways of bringing HTTP to a bundle network.

## HTTP Gateways {#use-gateways}

A pair of HTTP application-layer gateways, one at each edge of a bundle network, lets unmodified HTTP clients and origin servers communicate across it. Each gateway terminates the HTTP connections on its own side, and the gateways exchange requests and responses with each other using HTTP over BP. HTTP runs end to end, between client and origin server, while the transport beneath it changes from hop to hop:

~~~ artwork
+--------+                                                +--------+
|        |<==================== HTTP ====================>|        |
|        |                                                |        |
|        |      +-------------+      +-------------+      |        |
| client |<---->| client-side |<---->| origin-side |<---->| origin |
|        |  IP  |   gateway   |  BP  |   gateway   |  IP  | server |
+--------+      +-------------+      +-------------+      +--------+
~~~
{: #fig-overview title="HTTP Gateways at the Edges of a Bundle Network"}

The DTN architecture anticipated gateways of this kind. One of its stated aims is "to provide a common method for interconnecting application layer gateways and proxies", so that existing Internet applications that can be made to tolerate delay can "operate unmodified over a DTN-based network" ({{Section 3.11 of RFC4838}}). HTTP over BP is that common method for HTTP: the protocol beneath the web, most application programming interfaces, and most streaming media. With it, the large body of existing HTTP clients, servers, tools, and libraries can be used across bundle networks without being rewritten.

A gateway does more than translate between transports. HTTP's own means of coping with an expensive path work at the application layer, where only an intermediary can apply them: a client-side gateway answers what it can from its cache, collapses identical requests into one, and revalidates stored responses with conditional requests that cost little when nothing has changed ({{caching}}). Content can also be placed on the near side of a bundle network in advance, so that only cache fill crosses it. {{RFC3135}} describes such a device as an application layer performance enhancing proxy, with "the same functionality as the corresponding regular proxy for the same application (e.g., relay MTA or Web caching proxy) but extended with link-specific optimizations of the application protocol operation".

The design assumes disruption, not necessarily delay. On a bundle path with low latency the mechanisms described here for tolerating delay never engage: responses arrive before any client notices, and a gateway pair behaves as an ordinary caching reverse proxy.

## BP-Native HTTP {#use-native}

An application on a BP node, such as software on a spacecraft or a surface asset, can act as an HTTP client by sending requests over BP itself, with no IP stack and no client-side gateway. It can reach an origin server on the Internet through an origin-side gateway:

~~~ artwork
+------------------+                                      +--------+
|                  |<=============== HTTP ===============>|        |
|                  |                                      |        |
|                  |                 +-------------+      |        |
| BP-native client |<--------------->| origin-side |<---->| origin |
|                  |       BP        |   gateway   |  IP  | server |
+------------------+                 +-------------+      +--------+
~~~
{: #fig-native-client title="A BP-Native HTTP Client"}

An application can likewise act as an origin server directly over BP, publishing resources that clients on the Internet reach through a client-side gateway, and a BP-native client can reach a BP-native origin server with no gateway at all. Such applications use HTTP's methods, status codes, caching, and media types, and existing HTTP libraries for constructing and parsing messages, rather than defining an application protocol of their own. A node can choose for itself whether to reach a service over IP, through a gateway, or over BP directly, and the HTTP semantics are the same in each case.

## Alternatives {#alternatives}

There are other ways to bring HTTP to a bundle network, and each has a cost.

Application-specific protocols:
: Applications can be rewritten to use BP directly, with an application protocol of their own. This suits purpose-built applications, such as spacecraft operations, but cannot be asked of HTTP software in general.

End-to-end transport:
: TCP/IP or QUIC can be run end to end, either directly across the path or by carrying IP packets in bundles. These transports assume a contemporaneous end-to-end path: they spend round trips establishing and securing a connection, retransmit on short timers, and treat a path that stays silent for long as failed.

HTTP connections in bundles:
: The bytes of an HTTP connection can be carried in bundles {{I-D.blanchet-dtn-http-over-bp}}. This avoids IP, but inherits the connection's need for reliable, in-order delivery, which BP does not provide.

Carrying HTTP's messages, rather than its connections, avoids all of these costs.

# Architecture {#architecture}

HTTP is well suited to being carried as individual messages. Its semantics are defined independently of the protocols that carry them, and have already been mapped onto more than one transport, most recently QUIC {{RFC9114}}. Each request and each response is a self-contained message. HTTP already defines the role of intermediaries ({{Section 3.7 of RFC9110}}), which fields are end to end and which belong to a single connection ({{Section 7.6.1 of RFC9110}}), and how intermediaries cache {{RFC9111}}. This document therefore needs to specify little beyond the bundle hop.

## Roles

The protocol in this document operates between a requester and a responder. A requester is either a client-side gateway or a BP-native client; a responder is either an origin-side gateway or a BP-native origin server. The protocol is the same whichever kind each is, so any requester can exchange requests and responses with any responder. {{fig-overview}} shows a client-side gateway as the requester and an origin-side gateway as the responder; {{fig-native-client}} shows a BP-native client as the requester. A BP-native origin server answers requests for its own authorities directly:

~~~ artwork
+--------+                                      +--------------------+
|        |<=============== HTTP ===============>|                    |
|        |                                      |                    |
|        |      +-------------+                 |                    |
| client |<---->| client-side |<--------------->| BP-native origin   |
|        |  IP  |   gateway   |       BP        |       server       |
+--------+      +-------------+                 +--------------------+
~~~
{: #fig-native-origin title="A BP-Native HTTP Origin Server"}

Requirements in this document that apply to every requester or every responder are stated in those terms; requirements specific to gateways are stated in terms of client-side or origin-side gateways.

A single node can act in both roles, for example as a relay between two bundle networks, or as a gateway that serves clients in each direction ({{chained}}).

Gateways are HTTP intermediaries ({{Section 3.7 of RFC9110}}), and all of the requirements that {{RFC9110}} and {{RFC9111}} place on intermediaries apply to them. A client-side gateway commonly acts as a reverse proxy for the authorities it serves; it might also be configured as a forward proxy for "http" targets. In either case it terminates the client's HTTP connection, and TLS where used, and an origin-side gateway opens its own connection to the origin server. The bundle hop between them is protected with BPSec ({{security}}).

## Principles {#principles}

The mapping rests on four principles.

One message per bundle:
: Each ADU carries exactly one complete HTTP message. A receiver can interpret every ADU on its own, without reference to any other bundle.

Explicit correlation:
: How a response is correlated with its request depends on the HTTP version; some versions rely on the ordering of messages on a connection, and others use an explicit identifier ({{Section 7.5 of RFC9110}}). BP preserves no ordering, so this mapping uses an explicit correlation identifier carried in an envelope around the HTTP message ({{envelope}}).

No sequencing:
: No sequence numbers, acknowledgements, or retransmission state exist at this layer. Reordering is harmless because exchanges are matched by identifier, not by arrival order; duplicates are discarded because they match no outstanding exchange. Nor is ordering provided between distinct exchanges, just as HTTP provides none between requests sent on different connections.

Connections end at the edges:
: HTTP connections, and the TLS sessions that protect them, exist only between clients and client-side gateways and between origin-side gateways and origin servers. HTTP semantics are end to end; HTTP connections are not.

# Endpoints and Addressing {#addressing}

## Responder Endpoint {#service-endpoint}

A responder receives request ADUs at an endpoint registered for this service. The well-known endpoint is ipn-scheme service number TBD1 ({{iana-service}}) at the responder's node; for example, `ipn:977000.100.TBD1` is the responder endpoint of node 100 under allocator 977000 {{RFC9758}}.

A responder SHOULD support the well-known endpoint, so that requesters can reach it without additional configuration. A responder MAY also receive requests at other endpoints, which requesters must learn by other means.

A requester MUST address each request bundle to a singleton endpoint, because it relies on the response bundle being sourced from the same endpoint ({{receiving-response}}).

## Requester Endpoint

A requester sends each request bundle from a singleton endpoint at which it is registered, and receives the responses at that endpoint. A requester MUST NOT send a request bundle whose source is the null endpoint ID ({{Section 4.2.5.1.1 of RFC9171}}), since there would be no endpoint to which to return the response.

The requester's endpoint need not be the well-known service endpoint. A node that acts as both requester and responder MAY use one endpoint for both roles: the direction of each ADU is given by the Binary HTTP framing indicator ({{message}}).

## Selecting a Responder {#discovery}

To send a request, a requester maps the authority component of the target URI to the EID of a responder. The mapping can come from configuration, including a default responder for authorities with no specific entry, or from a resolution service, for example one built on DNS Service Binding records {{RFC9460}}. Specifying such a service is outside the scope of this document; whatever the mechanism:

- A specific mapping for an authority MUST take precedence over a default.
- The integrity of the mapping MUST be protected, since an attacker who can alter it can direct requests, and the responses that populate a shared cache, to a responder of its choosing ({{security}}).
- A name with local scope, such as a single-label name or one under a special-use domain, MUST NOT be resolved using global resolution services.

A requester that finds no responder for an authority MUST NOT send a request ADU. A client-side gateway in that situation SHOULD respond with 502 (Bad Gateway) and a Proxy-Status field {{RFC9209}} with the error type "destination_not_found".

# ADU Format {#format}

## Structure {#envelope}

Each ADU consists of exactly one CBOR {{RFC8949}} data item: an array of four elements.

~~~ cddl
http-over-bp-adu = [
  version: 1,
  correlation: correlation-id,
  options: options,
  message: bstr,          ; a Binary HTTP message (RFC 9292)
]

correlation-id = bstr .size (1..64)

options = {
  ? 1 => true,            ; incremental (requests)
  ? 2 => true,            ; final-segment (responses)
  ? -1 => uint,           ; segment (requests and responses)
  * int => any            ; options defined by future specifications
}
~~~
{: #fig-cddl title="ADU Structure"}

A sender MUST use definite-length encoding for every data item in the ADU and MUST NOT repeat a key within the options map; senders SHOULD use the core deterministic encoding of {{Section 4.2.1 of RFC8949}}. The ADU MUST contain no data after the array.

A receiver MUST discard, without responding, an ADU that is not well-formed CBOR, that does not match {{fig-cddl}}, that contains a duplicate map key, or that has trailing data.

## Version

The version element identifies the format of the remainder of the ADU; this document specifies version 1. A receiver MUST discard, without responding, an ADU with a version it does not support. The version sits outside the structure it versions, so that a future version can change any other element.

## Correlation Identifier {#correlation}

The correlation identifier associates a response with its request. The requester generates it; the responder treats it as opaque and copies it unchanged into each response ADU it sends for that request.

A requester MUST NOT use a correlation identifier that is already in use for another outstanding exchange from the same requester endpoint, except when retrying a request ({{sending-request}}). Identifiers SHOULD be at least 16 bytes long and generated by a cryptographically secure random number generator, so that an attacker cannot predict them ({{security}}). A requester MAY instead encode its own state in the identifier, for example an authenticated token that lets it process responses without retaining per-exchange state, provided the result is equally unpredictable.

## Options {#options}

The options element is a map of integer keys to values, carrying metadata about the exchange that is not part of the HTTP message. Keeping this metadata out of the HTTP header and trailer fields ensures that no client or origin server can inject it, and that it never leaks to them ({{rationale}}).

The sign of a key indicates whether the option is critical:

- An option with a non-negative key is elective. A receiver MUST ignore an elective option that it does not understand, or that is defined only for the other direction.
- An option with a negative key is critical. A responder that receives a request ADU containing a critical option that it does not understand MUST NOT process the request and SHOULD respond with 501 (Not Implemented) and a Proxy-Status field {{RFC9209}} whose "details" parameter identifies the option. A requester that receives a response ADU containing a critical option that it does not understand MUST discard the ADU and treat the exchange as failed.

Key 0 is reserved. The options defined by this document are listed in {{tab-options}}; further options are registered as described in {{iana-options}}.

| Key | Name          | Used in             | Value    | Reference         |
|----:|---------------|---------------------|----------|-------------------|
| 1   | incremental   | Requests            | true     | {{incremental}}   |
| 2   | final-segment | Responses           | true     | {{event-streams}} |
| -1  | segment       | Requests, responses | uint     | {{event-streams}} |
{: #tab-options title="Options Defined by This Document"}

The timing of an exchange is not carried in options: it is taken from the bundles that carry the exchange ({{bundle-times}}).

## HTTP Message {#message}

The message element contains one Binary HTTP message {{RFC9292}}. A sender MUST use the known-length format: framing indicator 0 for a request and 1 for a response ({{Section 3.3 of RFC9292}}). A receiver MAY treat a message in the indeterminate-length format as invalid; the entire message is present in the ADU, so that format offers no benefit here.

The framing indicator determines the direction of the ADU: an ADU whose message is a request is a request ADU, and one whose message is a response is a response ADU.

The request control data ({{Section 3.4 of RFC9292}}) carries the method and the scheme, authority, and path of the target URI. The authority MUST NOT be empty. A request MAY include a Host header field, in which case it MUST identify the same authority as the control data; a responder MUST treat a request in which they differ as malformed (compare {{Section 8.3.1 of RFC9113}}). A responder MUST use the authority in the control data, not the Host field, to identify the target.

A response message carries exactly one final response. A responder SHOULD NOT include informational (1xx) responses ({{Section 3.5.1 of RFC9292}}): they are hints about a connection that no longer exists by the time the response is delivered. A requester MUST ignore any that are present.

Messages MAY carry trailer fields, and MAY be padded ({{Section 3.8 of RFC9292}}). Padding obscures message length only when the ADU is also encrypted ({{privacy}}).

## Field Handling {#fields}

No HTTP connection spans the bundle hop, so fields that describe a connection have nothing to apply to. Before encoding a message, the sender MUST remove:

- the connection-specific fields listed in {{Section 7.6.1 of RFC9110}}: Connection and every field it names, Proxy-Connection, Keep-Alive, TE, Transfer-Encoding, and Upgrade. {{Section 3.6 of RFC9292}} recommends removing these; this document requires it;
- the proxy authentication fields of {{Section 11.7 of RFC9110}}, which apply only to the hop on which they were received; and
- fields consumed by the gateway itself, such as Client-Cert and Client-Cert-Chain {{RFC9440}}, which client credentials must never carry beyond the gateway that verified them.

A client-side gateway has to receive the complete request content before it can encode the request, so it satisfies an `Expect: 100-continue` expectation itself, sending either 100 (Continue) or a final response as {{Section 10.1.1 of RFC9110}} requires of a server, and removes the expectation from the request before encoding it.

Each gateway MUST record itself in the Via header field as {{Section 7.6.3 of RFC9110}} requires of an HTTP-to-HTTP gateway. A client-side gateway appends its entry to the request before encoding it; an origin-side gateway appends its entry before forwarding the request to the origin server. Each SHOULD likewise append its entry to the response. Gateways SHOULD use a pseudonym rather than a host name as the received-by value, as {{Section 7.6.3 of RFC9110}} permits.

> The received-protocol that an origin-side gateway records for a request received over BP is an open issue ({{open-issues}}).
{:cref}

A responder with a clock that receives a response without a Date header field from an origin server MUST add one before encoding the response, as {{Section 6.6.1 of RFC9110}} requires of any recipient that forwards such a response.

# Exchange Procedures {#procedures}

## Bundle Age and Expiry {#bundle-times}

The procedures in this section use two properties of each bundle that the BPA delivers: its bundle age, the time that has elapsed since it was created, and its expiry, the time at which its age will exceed its lifetime. {{Section 5.5 of RFC9171}} determines bundle age from the bundle's creation timestamp where that timestamp is not zero and the receiving node's clock is accurate, and otherwise from the Bundle Age block ({{Section 4.4.2 of RFC9171}}), so both properties are available whether or not the nodes involved have accurate clocks.

The timing of an exchange is carried by its bundles rather than in the ADU. The lifetime of the request bundle bounds the whole exchange ({{sending-request}}), the response bundle expires no later than the request bundle ({{sending-response}}), and the bundle age of a response measures the time it spent in transit ({{age}}).

{{RFC9171}} does not specify the interface between a BPA and the application agent. A requester or responder relies on its BPA to report the bundle age and expiry of each bundle it delivers.

## Sending a Request {#sending-request}

To send a request, a requester:

1. selects a responder for the target URI ({{discovery}});
2. constructs the request message as specified in {{message}} and {{fields}};
3. generates a correlation identifier ({{correlation}}) and constructs the request ADU;
4. requests transmission of a bundle whose payload is the request ADU, whose destination is the responder endpoint, and whose source is the requester endpoint ({{Section 5.2 of RFC9171}}); and
5. records the exchange: its correlation identifier, the responder endpoint to which it was sent, and its deadline ({{failure}}).

Each request ADU is sent in a bundle of its own. A requester MUST NOT place more than one ADU in a bundle. The BPA may fragment the bundle in transit ({{Section 5.8 of RFC9171}}); the responder's BPA reassembles it before delivery.

The lifetime of the request bundle bounds the whole exchange, since no response can be sent after the request bundle expires ({{sending-response}}). It SHOULD be long enough for the request to reach the responder and for the response to return, for example as estimated from a contact plan, and no longer than the period during which the requester will use a response. A requester MAY request bundle deletion status reports ({{Section 6.1.1 of RFC9171}}) so that it can detect failure early ({{failure}}); it SHOULD NOT request other status reports, except for diagnostics, because each report adds traffic to the bundle network.

A requester MUST NOT automatically retry a request whose method is not idempotent ({{Section 9.2.2 of RFC9110}}). It MAY retry a request whose method is safe or idempotent, for example after a deletion status report or once its deadline has passed. A retry SHOULD reuse the correlation identifier of the original request, so that a response to either attempt completes the exchange and a response to the other is discarded as a duplicate.

## Processing a Request {#processing-request}

When an ADU is delivered to a responder ({{Section 5.7 of RFC9171}}), the responder MUST discard it, without responding, if:

- the ADU is not valid as specified in {{envelope}}, or has an unsupported version;
- its message is a response;
- the source of the bundle that carried it is the null endpoint ID; or
- the bundle's "ADU is an administrative record" flag is set.

If the message is not a valid Binary HTTP message ({{Section 4 of RFC9292}}) or is malformed as described in {{message}}, the responder SHOULD respond with 400 (Bad Request). If the request contains a critical option the responder does not understand, it proceeds as specified in {{options}}.

Otherwise the responder handles the request. A BP-native origin server handles it as an origin server would. An origin-side gateway forwards it to the origin server as an HTTP gateway ({{Section 3.7 of RFC9110}}): it identifies the origin from the target URI, establishes a connection, verifying the server's certificate for an "https" target as {{Section 4.3.4 of RFC9110}} requires, applies the field handling of {{fields}}, and sends the request. A responder that has a cache MAY answer from a stored response under the rules of {{RFC9111}}. An origin-side gateway SHOULD apply a policy that restricts the destinations it will contact ({{ssrf}}).

A request bundle that expires before it is delivered is deleted by the BPA ({{Section 5.5 of RFC9171}}), so a responder does not process a request after its requester has stopped waiting for the response. A responder MAY also discard, without processing, a request that it estimates cannot be answered before the request bundle expires, for example from a contact plan.

A responder can receive the same request more than once: BP can duplicate bundles, and a requester can retry a request with its original correlation identifier. Re-executing a request whose method is not idempotent could repeat its effect. For such requests, a responder SHOULD recognise a duplicate by the source of the request bundle together with its correlation identifier and, rather than processing the request again, re-send the response it gave to the first copy, or nothing if that response is not yet complete. A responder that does so SHOULD retain its record of each exchange at least until the latest expiry of any request bundle it has received for that exchange ({{bundle-times}}).

## Sending a Response {#sending-response}

To answer a request, the responder constructs a response ADU with:

- version 1;
- the correlation identifier copied, unchanged, from the request ADU;
- any options that apply; and
- the final response, encoded as specified in {{message}} and {{fields}}.

It requests transmission of a bundle whose payload is the response ADU, whose destination is the source node ID of the request bundle, and whose source is the endpoint to which the request bundle was addressed. The responder MUST set the lifetime of the response bundle so that it expires no later than the request bundle ({{bundle-times}}), and SHOULD NOT send a response ADU after the request bundle has expired.

A responder sends one response ADU for each request it processes, except as specified in {{extended}}.

When an origin-side gateway cannot complete the exchange with the origin server, it responds with an error response such as 502 (Bad Gateway) or 504 (Gateway Timeout), carrying a Proxy-Status field {{RFC9209}} that identifies the gateway and the error type, for example "dns_error", "connection_refused", "connection_timeout", "tls_certificate_error", or "http_response_timeout". Clients can then distinguish a failure beyond the origin-side gateway from a failure in the bundle network ({{failure}}).

## Receiving a Response {#receiving-response}

When an ADU whose message is a response is delivered to a requester, the requester MUST discard it unless both:

- the requester has an outstanding exchange with the ADU's correlation identifier; and
- the source of the bundle that carried the ADU is the endpoint to which the request of that exchange was sent.

This check is the requester's defence against forged responses, which could otherwise poison a shared cache for every client of the gateway; its strength depends on the authentication of the bundle's source ({{admission}}).

A requester MUST discard a response ADU for an exchange that has already completed, except as specified in {{extended}}. Otherwise, the response completes the exchange. The requester processes the response as an HTTP intermediary or client would: a client-side gateway applies the field handling of {{fields}}, calculates the response's age ({{age}}), stores it if permitted ({{caching}}), and forwards it to the client, or retains it for the client to collect ({{client-facing}}).

## Deadlines and Failure {#failure}

A requester assigns each exchange a deadline, after which, if no response has been received, the exchange has failed. The deadline SHOULD be the expiry of the request bundle: no response can arrive after it, since a response bundle expires no later than the request bundle ({{sending-response}}).

When an exchange fails, a client-side gateway answers any client waiting for it with 504 (Gateway Timeout) and a Proxy-Status field with the error type "http_response_timeout".

A requester that receives a bundle status report ({{Section 6.1.1 of RFC9171}}) asserting that the request bundle was deleted MAY fail the exchange immediately. A client-side gateway that does so SHOULD answer with 504 (Gateway Timeout) and a Proxy-Status field with the error type "bp_bundle_deleted" ({{iana-proxy-status}}), whose "bp-reason" parameter carries the reason code from the status report.

As with any 504 response, the failure of an exchange does not show that the request was not processed: the response could have been lost or delayed after the responder acted on it.

After a safe request fails, a client-side gateway SHOULD NOT send the same request again until a configured interval has passed, so that repeated client requests do not send a stream of bundles toward an unreachable responder.

# Extended Exchanges {#extended}

## Incremental Responses {#incremental}

Some responses are produced slowly: an origin server sends the header section promptly, then content over an extended period. No responder can send content that has not yet been produced, and holding the response until it is complete delays every byte of it. A requester that is able to assemble a response from parts indicates so by including the incremental option (key 1) in its request ADU.

A responder that receives a request with the incremental option MAY return the response as a sequence of 206 (Partial Content) responses ({{Section 15.3.7 of RFC9110}}), each in its own response ADU with the request's correlation identifier. Each of these responses:

- is complete and self-contained, with the full header section of the response and a single Content-Range header field ({{Section 14.4 of RFC9110}}) covering exactly the content it carries;
- carries the same strong validator ({{Section 8.8.1 of RFC9110}}) as every other response in the sequence; and
- carries the complete length of the representation in its Content-Range field if that length is known when it is sent, and "*" otherwise.

The responder chooses when to send each part. A response that is complete before the responder would send its first part is sent as a single ordinary response.

The requester combines the parts as {{Section 3.3 of RFC9111}} and {{Section 3.4 of RFC9111}} allow a cache to combine partial content; their order of arrival is immaterial. The exchange remains open until the requester holds the complete representation or its deadline passes; a requester that asks for an incremental response sets the lifetime of the request bundle to cover the time the response will take to complete. Once the complete length is known, the requester can recover any missing range with an ordinary range request in a new exchange. A requester MUST NOT serve the combined representation as complete until it is complete; it MAY answer a client's range request from content already received ({{Section 3.3 of RFC9111}}).

A responder SHOULD bound the resources it spends on any one incremental response, such as bytes, parts, and elapsed time. A responder that abandons a response before it is complete SHOULD send a final response ADU containing a 502 (Bad Gateway) response with a Proxy-Status field (for example, with error type "http_response_body_size"). On receiving it, the requester fails the exchange and MUST NOT treat the content received so far as a complete representation.

> An origin-side gateway might need to supply a strong validator for a response that has none. Whether, and how, it should do so without creating a validator the origin server would not recognise is an open issue ({{open-issues}}).
{:cref}

## Event Streams {#event-streams}

An event stream (the `text/event-stream` media type {{HTML}}) has no end, so it can be carried neither as a single response nor as a finite sequence of parts. Its contract is nevertheless tolerant of delay: events are delivered in order, in one direction, eventually, and a client can resume a stream from the identifier of the last event it received. This document carries an event stream as a sequence of finite segments.

> This section is less mature than the rest of the document, and could move to a companion document.
{:cref}

To request segment K of an event stream, a requester sends a GET request for the stream's target URI with the segment option (key -1) set to K. The option is critical, so a responder that does not support segments refuses the request ({{options}}) instead of opening a response that never ends.

An origin-side gateway that supports segments:

- maintains one subscription to the origin server's event stream for each target URI and credential scope, shared by every requester that requests segments of it;
- divides the events it receives into numbered segments at boundaries of its choosing, such as a count of events, a size, or an interval, and never splits an event across segments;
- answers a request for segment K, once that segment is closed, with a 200 (OK) response of type `text/event-stream` whose content is the complete events of the segment, in a response ADU carrying the segment option set to K;
- holds a request for a segment that is still open until the segment closes, and closes a segment early, even if it holds no events, so that its response can be delivered before the request bundle expires; and
- marks the last segment, when the origin server ends the stream, with the final-segment option (key 2).

A closed segment never changes, so a requester can store it and serve it to any number of local clients. A client-side gateway serves each client a genuine `text/event-stream` response on a connection that it holds open, releasing events strictly in order; a missing segment holds back later ones until it is recovered. The gateway requests further segments while it has clients for the stream, and MAY request segment K+1 before segment K arrives, to hide the latency of the path. It maps a client's Last-Event-ID to a position within the segments it holds.

Other unbounded response types, such as newline-delimited records, could be segmented in the same way, given a rule for event boundaries; such types do not identify themselves, so a gateway would need configuration to recognise them.

# Unsupported HTTP Features {#unsupported}

The CONNECT method ({{Section 9.3.6 of RFC9110}}) establishes a tunnel whose contents are interactive and opaque to HTTP. A requester MUST NOT send a CONNECT request in a request ADU. A client-side gateway that receives a CONNECT request asking it to establish a tunnel across the bundle network SHOULD respond with 501 (Not Implemented).

Protocol upgrade ({{Section 7.8 of RFC9110}}) replaces HTTP on a connection, and no connection spans the bundle hop. A client-side gateway that receives a request to upgrade to WebSocket {{RFC6455}}, whether through the Upgrade field or through extended CONNECT {{RFC8441}} {{RFC9220}}, SHOULD respond with 501 (Not Implemented) rather than forward it with the Upgrade field removed, which would return a response that the client did not ask for.

Applications that need messages in both directions can use an event stream for one direction ({{event-streams}}) and individual requests for the other.

# Gateway Considerations {#gateways}

This section gives requirements and guidance for gateways. None of it affects the format of ADUs or the exchange procedures, and gateways that make different choices interoperate.

## Caching {#caching}

Much of the value of a client-side gateway lies in its cache: every request it answers locally is one that does not cross the bundle network. A client-side gateway SHOULD act as a shared cache {{RFC9111}}. A BP-native client benefits from a private cache in the same way, and the guidance in this section applies to it as well, except where it concerns shared caches or requests from multiple clients.

Collapsing requests:
: A client-side gateway SHOULD collapse requests ({{Section 4 of RFC9111}}): when a request could be satisfied by the response to a safe request for which an exchange is already outstanding, the gateway awaits that response instead of starting another exchange. The key on which requests are collapsed MUST distinguish requests made with different credentials, whether carried in Authorization or Cookie fields or otherwise, and SHOULD include the request fields to which the response is expected to vary. A gateway MUST NOT collapse unsafe requests.

Validation:
: When it revalidates a stored response, a gateway SHOULD send a conditional request carrying the stored validators, so that an unchanged resource costs only a small 304 (Not Modified) response ADU. A gateway SHOULD answer a client's conditional request from a stored response wherever {{Section 4.3.2 of RFC9111}} allows, without starting an exchange.

Serving stale responses:
: {{Section 4.2.4 of RFC9111}} permits a cache to serve a stale response where it is explicitly permitted "by configuration in accordance with an out-of-band contract". A gateway deployed in front of a constrained or intermittent path is such a configuration, and MAY serve stale responses within configured limits, for example while revalidating in the background or after an exchange has failed (compare {{RFC5861}}). Nothing in this document relaxes the response directives of {{Section 5.2.2 of RFC9111}}: a gateway MUST NOT serve a stale response that a directive such as no-cache, must-revalidate, proxy-revalidate, or s-maxage prohibits, and MUST generate an accurate Age header field whenever it serves a stored response without validation.

Request directives:
: Cache request directives are advisory ({{Section 5.2.1 of RFC9111}}). A gateway MAY decline to honour directives, such as max-age, min-fresh, and no-cache, that would oblige it to start an exchange across the bundle network. It SHOULD honour only-if-cached, max-stale, and no-store, which cost nothing on the path.

Invalidation:
: On receiving a non-error response to an unsafe request, a gateway invalidates stored responses as {{Section 4.4 of RFC9111}} requires.

Pacing:
: A gateway MAY enforce a minimum interval between exchanges for the same resource. When it does so, and no stored response can be served, it responds with 504 (Gateway Timeout), which it MAY accompany with a Retry-After field ({{Section 10.2.3 of RFC9110}}) giving the time at which a new exchange will be allowed. Pacing never extends the circumstances in which a stored response may be served.

## Calculating Age {#age}

A requester that stores or forwards a response calculates its age as {{Section 4.2.3 of RFC9111}} specifies. That calculation's corrected_age_value adds to the received Age value the response_delay: the time between sending the request and receiving the response, the full round trip. Over a bundle path that round trip can be long, and much of it elapses before the response exists: the request's transit, and any time it waited for a contact. Charging all of it against the response's freshness lifetime roughly halves the usable lifetime on a symmetric path, and on a long-delay path can make every response stale on arrival.

The bundle age of the response bundle ({{bundle-times}}), as reported by the requester's BPA when the response is delivered, is the time that has elapsed since the responder requested its transmission: the time the response spent in transit. The requester MAY use it as transit_delay in place of response_delay:

~~~ pseudocode
corrected_age_value = age_value + transit_delay
~~~

The result is the response's age when the responder sent it, plus the time it has since spent in transit. A responder that does not send a response as soon as it has it, for example because it answers from a stored response or re-sends a response to a duplicate request, accounts for the delay in the Age field that it sends, as any cache does ({{Section 5.1 of RFC9111}}).

The apparent_age calculation of {{RFC9111}} would also exclude the request's transit, but relies on the origin server's clock, in the Date field, being synchronised with the requester's. A gateway cannot know whether an origin server's clock is synchronised. Bundle age is instead determined by the BPAs on the path, from the creation timestamp where the clocks of the responder's and requester's nodes are accurate, or from the Bundle Age block, which needs no synchronised clocks ({{timing}}).

A requester whose BPA does not report the bundle age of a response uses response_delay as {{RFC9111}} specifies. A requester MUST NOT treat a response as having an age of zero when it arrives; that would conceal the transit delay that the Age field exists to disclose.

## Responding to Clients During an Exchange {#client-facing}

A client expects a timely response, but over a long-delay path an exchange can outlast the client's patience. How a client-side gateway answers a client while an exchange is outstanding is local to the gateway. The following strategies are available, and gateways can combine them.

Holding:
: The gateway holds the client's connection open until the response arrives, or the exchange fails, and then responds normally. Holding is transparent whenever the round trip is shorter than the client's timeout, and is the only safe choice for programmatic clients, which would misinterpret anything else.

Polling in place:
: For a top-level navigation by a browser, the gateway can respond with 202 (Accepted), a Retry-After field, and a document that refreshes itself, so that the browser requests the same URI again later and receives the stored response once it has arrived. Because a programmatic client treats any 2xx response as success, and would consume the 202 response body as though it were the resource, a gateway SHOULD respond with 202 only when the request is known to be a navigation, for example from the Sec-Fetch-Dest request field {{FETCH-METADATA}}, and not on the strength of the Accept field.

Status resources:
: For an unsafe request, the gateway can respond with 303 (See Other) referring to a status resource, under an unguessable URI, from which the client retrieves the eventual response. A client can ask for this behaviour with `Prefer: respond-async` {{RFC7240}}, a preference that a proxy may honour on its own ({{Section 2 of RFC7240}}). The Idempotency-Key field {{I-D.ietf-httpapi-idempotency-key-header}} lets a client resubmit an unsafe request without repeating its effect.

Retry intervals:
: Where a gateway can estimate when a response will arrive, for example from a contact plan, it SHOULD derive the intervals it gives clients in Retry-After fields from that estimate, rather than from a fixed value.

Clients that send large or open-ended request content can divide it into finite steps with Resumable Uploads for HTTP {{I-D.ietf-httpbis-resumable-upload}}, each step being an ordinary exchange.

## Ordering

No ordering is provided between exchanges ({{principles}}). Two unsafe requests sent in succession can reach the origin server in either order. A client that needs one request to be processed after another has to wait for the response to the first before sending the second.

## Chained Gateways {#chained}

A node can act as both responder and requester, receiving requests over one bundle network and forwarding them over another. Each such node is an intermediary in its own right, and records itself in the Via field ({{fields}}). A misconfigured chain can form a loop; a gateway SHOULD detect loops, using its Via entries or the CDN-Loop field {{RFC8586}}, and respond with 502 (Bad Gateway) and a Proxy-Status field with the error type "proxy_loop_detected".

# Operational Considerations

Lifetimes and deadlines:
: Bundle lifetimes, and so exchange deadlines, need to match the latency of the path. Contact plans, where available, give the best estimates. Values that are too short fail exchanges that would have completed; values that are too long hold resources for responses that will never be used.

Clocks:
: Exchange deadlines and age calculation ({{age}}) depend on the BPAs' determination of bundle age. BP nodes generally keep accurate time; a deployment with clockless nodes relies on the Bundle Age block, which every node on the path then needs to maintain.

Message size:
: Each HTTP message is a single ADU, carried and if necessary fragmented by the BP layer. Deployments typically limit message sizes. A client-side gateway that refuses request content as too large responds with 413 (Content Too Large); a responder that refuses a response as too large responds with 502 (Bad Gateway) and a Proxy-Status field with the error type "http_response_body_size".

Round trips:
: Some HTTP patterns cost a round trip each: redirects, authentication challenges, and preflight requests. Over a long-delay path each of these can cost hours. Origins that expect to be reached across a bundle network can avoid them, for example by sending credentials preemptively.

Time-limited authorisations:
: Signed URIs, tokens, and licence exchanges that expire within less than the latency of the path expire in transit. This is a property of the origin's security design, which no gateway can work around.

Low-latency paths:
: On a path with low latency, requests complete before any client times out, and a gateway pair behaves like an ordinary caching reverse proxy. Nothing in the mapping needs to be configured differently.

# Security Considerations {#security}

## Trust Model

TLS terminates at the gateways, so there is no end-to-end TLS between client and origin server. A client entrusts the client-side gateway with the content of its requests and responses, as it would any reverse proxy that terminates TLS, and a gateway presents certificates only for names it is authorised to serve. The origin-side gateway authenticates the origin server ({{Section 4.3.4 of RFC9110}}). Intercepting TLS connections to arbitrary origins by means of a locally installed certification authority is outside the scope of this document; it exposes all traffic to the gateway, defeats certificate pinning, and is appropriate only where the operator controls the clients.

Between BP-native endpoints, and between a BP-native endpoint and a gateway, there is no TLS at all: BPSec is the only protection on the bundle hop ({{protecting-the-bundle-hop}}), and the authenticated source of a bundle takes the place of the TLS server identity. A BP-native client trusts the responder it has selected for an authority ({{discovery}}) as it would trust an origin server whose certificate it had verified, and needs that selection to be equally well protected.

## Protecting the Bundle Hop {#protecting-the-bundle-hop}

Requesters and responders MUST support BPSec {{RFC9172}} with the default security contexts {{RFC9173}}, and SHOULD protect every ADU with a Block Integrity Block (BIB) and a Block Confidentiality Block (BCB) targeting the payload block. The BIB SHOULD include the primary block in its integrity scope (the "Include primary block" flag of {{Section 3.3.3 of RFC9173}}), so that the source of the bundle, on which response admission depends, and its creation timestamp and lifetime, which bound the exchange, are authenticated. Without BPSec, an attacker on the path can read requests and responses, including credentials, and can forge responses. A deployment that relies on protection at another layer should document that dependency.

## Response Admission and Cache Poisoning {#admission}

A requester accepts a response only for an outstanding exchange, and only from the endpoint to which it sent the request ({{receiving-response}}). Where the source of the bundle is authenticated, no third party can inject a response. A forged response accepted by a client-side gateway could otherwise be stored in its shared cache and served to every client. The parts of an incremental response must also share a strong validator ({{incremental}}).

## Resolution Integrity

Response admission is only as strong as the mapping from authority to responder ({{discovery}}). An attacker that can alter the mapping can direct requests, and the responses that populate a shared cache, to a responder of its choosing. Mappings obtained from DNS need to be validated with DNSSEC; where that is not possible, mappings need to be configured or pinned.

## Correlation Identifiers

Unpredictable correlation identifiers ({{correlation}}) make blind injection of responses harder where the bundle hop is not protected. They are not a substitute for BPSec.

## Server-Side Request Forgery {#ssrf}

An origin-side gateway fetches whatever a request names, which makes it a means of reaching hosts on its own network. An origin-side gateway SHOULD refuse, unless configured otherwise, to contact private, loopback, and link-local addresses, including cloud metadata services, and SHOULD support restricting the origins it will contact to a configured set. It needs to apply these checks to the addresses that names resolve to, not only to literal addresses in the target URI. A gateway that refuses a request because of the address it would contact responds with 502 (Bad Gateway) and a Proxy-Status field {{RFC9209}} with the error type "destination_ip_prohibited"; one that refuses it because the origin is not in its configured set responds with 403 (Forbidden) and the error type "http_request_denied".

## Replay {#replay}

The default security contexts of BPSec {{RFC9173}} do not prevent replay. A replayed request bundle carrying an unsafe request could cause the request to be acted on again. The responder's detection of duplicate requests ({{processing-request}}) suppresses replays while its records are retained, and a replayed bundle expires when the original does, since its creation timestamp and lifetime are in the primary block, which the BIB protects. Together, these bound the window in which a replay can have any effect. The bound does not hold for a bundle whose creation time is zero: its age is carried in the Bundle Age block, which each forwarding node updates and so cannot usefully be covered by a BIB added at the source, and a replaying node can reset it. A requester SHOULD send each request whose method is not idempotent in a bundle with a non-zero creation timestamp, and with a lifetime no longer than the exchange needs.

## Timing Integrity {#timing}

The bundle age of a response determines the age that a requester computes for it ({{age}}), and so how long the response is served. An attacker that could alter it could make a response appear fresher, or more stale, than it is. Where the response bundle has a non-zero creation timestamp, that timestamp is in the primary block, which the BIB protects. Where its creation time is zero, the age is taken from the Bundle Age block, which each forwarding node updates and which a BIB added at the source cannot usefully cover; a deployment that relies on clockless nodes trusts the nodes on the path to maintain it honestly.

## Resource Exhaustion

A small request can elicit a large response, and an incremental response or an event stream can elicit a long sequence of them. Responders need to bound the resources any one request can consume, and both roles need to bound the number of outstanding exchanges, held client connections, and event-stream subscriptions. Requests for bundle status reports add traffic to the bundle network and should be limited accordingly.

## Credentials and Privacy {#privacy}

Authorization and Cookie fields cross the bundle network inside ADUs and need the confidentiality that a BCB provides. Client certificates are verified at the client-side gateway and never forwarded ({{fields}}). A shared cache does not store responses to requests with Authorization fields unless the origin server permits it ({{Section 3.5 of RFC9111}}), and gateways do not collapse requests made with different credentials ({{caching}}). Via pseudonyms ({{fields}}) avoid disclosing the topology of gateway deployments. Padding Binary HTTP messages ({{message}}) can obscure their lengths from an observer when the ADU is encrypted.

# IANA Considerations

## Well-Known Service Number {#iana-service}

This document requests that IANA register the following entry in the "'ipn' Scheme URI Well-Known Service Numbers for BPv7" registry established by {{RFC9758}}, assigning a value from the Standards Action range (128-255):

| Value | Description  | Reference       |
|-------|--------------|-----------------|
| TBD1  | HTTP over BP | (this document) |
{: #iana-service-tab align="left" title="Service Number Registration"}

> RFC Editor: please replace TBD1 throughout this document with the value assigned by IANA, and remove this note.
{:aside}

## HTTP over BP Options Registry {#iana-options}

This document requests that IANA create a new registry titled "HTTP over BP Options". Each entry records:

Key:
: An integer. Negative keys denote critical options and non-negative keys elective options ({{options}}).

Name:
: A short descriptive name.

Used in:
: Requests, responses, or both.

Value:
: The type of the value, in CDDL.

Reference:
: The specification that defines the option.

The registration procedures ({{RFC8126}}) are:

| Range                                       | Registration Procedure |
|---------------------------------------------|------------------------|
| -24 to 23                                   | Standards Action       |
| -65536 to -25, and 24 to 65535              | Specification Required |
| Less than -65536, and greater than 65535    | Private Use            |
{: #iana-options-policy align="left" title="HTTP over BP Options Registration Procedures"}

The initial contents of the registry are the options in {{tab-options}}, each with this document as the reference, together with key 0, marked "Reserved".

## HTTP Proxy Error Type {#iana-proxy-status}

This document requests that IANA register the following entry in the "HTTP Proxy Error Types" registry established by {{RFC9209}}:

Name:
: bp_bundle_deleted

Description:
: The intermediary was notified by the Bundle Protocol that a bundle carrying the request was deleted before it was delivered, for example because its lifetime expired or no route to its destination was available.

Extra Parameters:
: bp-reason: an Integer containing the reason code from the bundle status report ({{Section 6.1.1 of RFC9171}}).

Recommended HTTP Status Code:
: 504

Response Only Generated by Intermediaries:
: true

Reference:
: (this document)

--- back

# Examples {#examples}

This appendix shows one exchange in full. A client-side gateway, with Via pseudonym "gw-a" and endpoint `ipn:977000.1.4096`, forwards a client's request for `https://example.com/status.json` to an origin-side gateway, with pseudonym "gw-b" and endpoint `ipn:977000.2.TBD1`. The path takes 20 minutes in each direction.

## Request

The client-side gateway sends the request at 11:40:00 UTC on 23 October 2026. It will wait for the response until 12:40:00, so it gives the request bundle a lifetime of 60 minutes. The request message is:

~~~ artwork
Method:    GET
Scheme:    https
Authority: example.com
Path:      /status.json
Fields:    accept: application/json
           via: 1.1 gw-a
Content:   (empty)
~~~

The request ADU, in CBOR extended diagnostic notation ({{Appendix G of RFC8610}}), is:

~~~ cbor-diag
[
  1,                                    / version /
  h'00112233445566778899aabbccddeeff',  / correlation /
  {},                                   / options /
  h'00034745 54056874 7470730b 6578616d
    706c652e 636f6d0c 2f737461 7475732e
    6a736f6e 25066163 63657074 10617070
    6c696361 74696f6e 2f6a736f 6e037669
    6108312e 31206777 2d610000'         / message /
]
~~~

The 98 bytes of the encoded ADU are:

~~~ artwork
84015000 11223344 55667788 99aabbcc  ..P.."3DUfw.....
ddeeffa0 584c0003 47455405 68747470  ....XL..GET.http
730b6578 616d706c 652e636f 6d0c2f73  s.example.com./s
74617475 732e6a73 6f6e2506 61636365  tatus.json%.acce
70741061 70706c69 63617469 6f6e2f6a  pt.application/j
736f6e03 76696108 312e3120 67772d61  son.via.1.1 gw-a
0000                                 ..
~~~

## Response

The request is delivered at 12:00:00. The origin-side gateway appends its own Via entry, forwards the request to the origin server, and receives the response at 12:00:00.250. It appends its Via entry to the response, and sends it at once, in a bundle whose lifetime of 2399.75 seconds makes it expire with the request bundle at 12:40:00. The response message is:

~~~ artwork
Status:  200
Fields:  date: Fri, 23 Oct 2026 12:00:00 GMT
         cache-control: max-age=3600
         content-type: application/json
         etag: "v42"
         via: 1.1 gw-b
Content: {"status":"ok"}
~~~

The response ADU is:

~~~ cbor-diag
[
  1,                                    / version /
  h'00112233445566778899aabbccddeeff',  / correlation /
  {},                                   / options /
  h'0140c840 74046461 74651d46 72692c20
    3233204f 63742032 30323620 31323a30
    303a3030 20474d54 0d636163 68652d63
    6f6e7472 6f6c0c6d 61782d61 67653d33
    3630300c 636f6e74 656e742d 74797065
    10617070 6c696361 74696f6e 2f6a736f
    6e046574 61670522 76343222 03766961
    08312e31 2067772d 620f7b22 73746174
    7573223a 226f6b22 7d00'             / message /
]
~~~

The 160 bytes of the encoded ADU are:

~~~ artwork
84015000 11223344 55667788 99aabbcc  ..P.."3DUfw.....
ddeeffa0 588a0140 c8407404 64617465  ....X..@.@t.date
1d467269 2c203233 204f6374 20323032  .Fri, 23 Oct 202
36203132 3a30303a 30302047 4d540d63  6 12:00:00 GMT.c
61636865 2d636f6e 74726f6c 0c6d6178  ache-control.max
2d616765 3d333630 300c636f 6e74656e  -age=3600.conten
742d7479 70651061 70706c69 63617469  t-type.applicati
6f6e2f6a 736f6e04 65746167 05227634  on/json.etag."v4
32220376 69610831 2e312067 772d620f  2".via.1.1 gw-b.
7b227374 61747573 223a226f 6b227d00  {"status":"ok"}.
~~~

## Age {#example-age}

The response is delivered to the client-side gateway at 12:20:00.250. The response carries no Age field, so age_value is 0.

The client-side gateway's BPA reports the response bundle's age as 1200 seconds. Using it as transit_delay ({{age}}), so the response's age on arrival is 1200 seconds and it remains fresh for a further 2400 seconds.

Using the response_delay of {{RFC9111}}, which here is the full round trip of about 2400 seconds, the response's age on arrival would be 2400 seconds, and it would remain fresh for only 1200 seconds. The difference is the 20 minutes the request spent in transit, before the response existed.

# Design Rationale {#rationale}

Why Binary HTTP:
: Binary HTTP {{RFC9292}} represents a complete HTTP message independently of any HTTP version or connection, which is exactly what one message per bundle requires. It is compact, already deployed, and designed to be encapsulated.

Why an envelope rather than header fields:
: The correlation identifier and options could have been carried as HTTP fields inside the Binary HTTP message. They are kept in a separate envelope so that no client or origin server can set them, and so that gateways need not strip them before forwarding messages. The envelope can also carry a version outside the structure it versions.

Why no timing options:
: The lifetime of the request bundle bounds the exchange, and the bundle age of the response measures its transit ({{bundle-times}}). Both are carried by the bundle, determined by the BPA whether or not the nodes have accurate clocks, and protected with the primary block. Repeating them in the ADU would add fields that need clocks and that could disagree with the bundle's own.

Why not Oblivious HTTP:
: Oblivious HTTP also encapsulates Binary HTTP, but adds encryption under a separate key infrastructure. BPSec already protects the bundle hop, using the keys the bundle network provisions.

Why no sequence numbers:
: Every ADU is self-contained, exchanges are matched by identifier, and parts of incremental responses are placed by their Content-Range. Nothing depends on the order of arrival, so there is nothing for sequence numbers to repair.

Why gateways rather than tunnels:
: A gateway can answer requests from its cache, collapse requests, and revalidate cheaply, which are the means by which HTTP copes with an expensive path. A tunnel can do none of these things, and needs the interactive end-to-end connection that a bundle network cannot provide.

# Open Issues {#open-issues}
{:removeinrfc}

- Via received-protocol: which received-protocol an origin-side gateway should record for a request received over BP. Candidates include the client's HTTP version, which is not currently conveyed, or a new protocol name.
- dtn-scheme endpoints: whether to define a well-known dtn-scheme demux for this service, alongside the ipn service number.
- Validators for incremental responses: whether an origin-side gateway may supply a strong validator for a response that has none, or whether parts should instead be tied together by an envelope option.
- Event streams: how a requester joins a stream that is already in progress; how it learns that a responder's subscription has restarted and its segment numbering with it; how long responders retain closed segments, and how they answer requests for discarded ones; how streams answering unsafe requests are segmented; and whether this belongs in a companion document.
- Unknown critical options in requests: whether 501 (Not Implemented) is the right response.
- BPSec: whether its use should be required rather than recommended, and which security contexts implementations must support.
- Age calculation: substituting transit_delay for response_delay ({{age}}) relaxes a requirement of {{Section 4.2.3 of RFC9111}}, which says a cache "MUST interpret this value relative to the time the request was initiated". Whether this document needs to update RFC 9111, or can justify the calculation in some other way, needs review by HTTP caching experts.
- BPA interface: this document relies on the BPA reporting the bundle age and expiry of each delivered bundle ({{bundle-times}}), which {{RFC9171}} does not require of it.
- Media type: whether ADUs need a media type for storage or for transport outside BP.
- Naming: the name "HTTP over BP" was used by {{I-D.blanchet-dtn-http-over-bp}}, which specified a different approach.

# Acknowledgments
{:unnumbered}

TODO acknowledge.
