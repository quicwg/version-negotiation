---
title: Compatible Version Negotiation for QUIC
abbrev: QUIC Compatible VN
docname: draft-ietf-quic-version-negotiation-latest
category: std
wg: QUIC

ipr: trust200902
area: General
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: "D. Schinazi"
    name: "David Schinazi"
    organization: "Google LLC"
    street: "1600 Amphitheatre Parkway"
    city: "Mountain View, California 94043"
    country: "United States of America"
    email: dschinazi.ietf@gmail.com
  -
    ins: E. Rescorla
    name: Eric Rescorla
    organization: Mozilla
    email: ekr@rtfm.com

normative:
  INV: RFC8999
  QUIC: RFC9000


--- abstract

QUIC does not provide a complete version negotiation mechanism but instead only
provides a way for the server to indicate that the version the client offered is
unacceptable. This document describes a version negotiation mechanism that
allows a client and server to select a mutually supported version. Optionally,
if the original and negotiated version share a compatible first flight format,
the negotiation can take place without incurring an extra round trip.


--- middle

# Introduction

The version-invariant properties of QUIC {{INV}} define a version negotiation
(VN) packet but do not specify how an endpoint reacts when it receives one. QUIC
version 1 {{QUIC}} allows the server to use a VN packet to indicate that the
version the client offered is unacceptable, but doesn't allow the client to
safely make use of that information to create a new connection with a mutually
supported version.

With proper safety mechanisms in place, the VN packet can be part of a mechanism
to allow two QUIC implementations to negotiate between two totally disjoint
versions of QUIC. This document specifies version negotiation using VN packets,
which adds an extra round trip to connection establishment if needed.

It is beneficial to avoid additional round trips whenever possible, especially
given that most incremental versions are broadly similar to the the previous
version. This specification also defines a simple version negotiation mechanism
which leverages similarities between versions and can negotiate between the set
of "compatible" versions without additional round trips.


## Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

In this document, the Maximum Segment Lifetime (MSL) represents the time a QUIC
packet can exist in the network. Implementations can make this configurable, and
a RECOMMENDED value is one minute.


# Server Deployments of QUIC {#server-fleet}

While this document mainly discusses a single QUIC server, it is common for
deployments of QUIC servers to include a fleet of multiple server instances. We
therefore define the following terms:

Acceptable Versions:

: This is the set of versions supported by a given server instance. More
specifically, these are the versions that a given server instance will use if a
client sends a first flight using them.

Offered Versions:

: This is the set of versions that a given server instance will send in a
Version Negotiation packet if it receives a first flight from an unknown
version. This set will most often be equal to the Acceptaple Versions set,
except during short transitions while versions are added or removed (see below).

Fully-Deployed Versions:

: This is the set of QUIC versions that is supported and negotiated by every
single QUIC server instance in this deployment. If a deployment only contains a
single server instance, then this set is equal to the Offered Versions set,
except during short transitions while versions are added or removed (see below).

If a deployment contains multiple server instances, software updates may not
happen at exactly the same time on all server instances. Because of this, a
client might receive a Version Negotiation packet from a server instance that
has already been updated and the client's resulting connection attempt might
reach a different server instance which hasn't been updated yet.

However, even when there is only a single server instance, it is still possible
to receive a stale Version Negotiation packet if the server performs its
software update while the Version Negotiation packet is in flight.

This could cause the version downgrade prevention mechanism described in
{{downgrade}} to falsely detect a downgrade attack. To avoid that, server
operators SHOULD perform a three-step process when they wish to add or remove
support for a version:

When adding support for a new version:

* The first step is to progressively add support for the new version to all
  server instances. This step updates the Acceptable Versions but not the
  Offered Versions nor the Fully-Deployed Versions. Once all server instances
  have been updated, operators wait for at least one MSL to allow any in-flight
  Version Negotiation packets to arrive.

* Then, the second step is to progressively add the new version to Offered
  Versions on all server instances. Once complete, operators wait for at least
  another MSL.

* Finally, the third step is to progressively add the new version to
  Fully-Deployed Versions on all server instances.

When removing support for a version:

* The first step is to progressively remove the version from Fully-Deployed
  Versions on all server instances. Once it has been removed on all server
  instances, operators wait for at least one MSL to allow any in-flight Version
  Negotiation packets to arrive.

* Then, the second step is to progressively remove the version from Offered
  Versions on all server instances. Once complete, operators wait for at least
  another MSL.

* Finally, the third step is to progressively remove support for the version
  from all server instances. That step updates the Acceptable Versions.


Note that this opens connections to version downgrades (but only for
partially-deployed versions) during the update window, since those could be due
to clients communicating with both updated and non-updated server instances.


# Compatible Versions

If A and B are two distinct versions of QUIC, A is said to be "compatible" with
B if it is possible to take a first flight of packets from version A and convert
it into a first flight of packets from version B. As an example, if versions A
and B are absolutely equal in their wire image and behavior during the handshake
but differ after the handshake, then A is compatible with B and B is compatible
with A.

Version compatibility is not symmetric: it is possible for version A to be
compatible with version B and for B not to be compatible with A. This could
happen for example if version B is a strict superset of version A.

Note that version compatibility does not mean that every single possible
instance of a first flight will succeed in conversion to the other version. A
first flight using version A is said to be "compatible" with version B if two
conditions are met: first that version A is compatible with version B, and
second that the conversion of this first flight to version B is well-defined.
For example, if version B is equal to A in all aspects except it introduced a
new frame in its first flight that version A cannot parse or even ignore, then B
could still be compatible with A as conversions would succeed for connections
where that frame is not used. In this example, first flights using version B
that carry this new frame would not be compatible with version A.

When a new version of QUIC is defined, it is assumed to not be compatible with
any other version unless otherwise specified. Similarly, no other version is
compatible with the new version unless otherwise specified. Implementations MUST
NOT assume compatibility between versions unless explicitly specified.

Note that both endpoints might disagree on whether two versions are compatible
or not. For example, two versions could have been defined concurrently and then
specified as compatible in a third document much later - in that scenario one
endpoint might be aware of the compatibility document while the other may not.

When a client creates a QUIC connection, its goal is to use an application layer
protocol. Therefore, when considering which versions are compatible, clients
will only consider versions that support the intended application layer protocol.


# Version Negotiation Mechanism

This document specifies two means of performing version negotiation: one
"incompatible" which requires a round trip and is applicable to all versions,
and one "compatible" that allows saving the round trip but only applies when the
versions are compatible.

The client initiates a QUIC connection by sending a first flight of QUIC packets
with a long header to the server {{INV}}. We'll refer to the version of those
packets as the "original version". The client's first flight includes Version
Information (see {{vers-info}}) which will be used to optionally enable
compatible version negotation (see {{compat-vn}}), and to prevent version
downgrade attacks (see {{downgrade}}).

Upon receiving this first flight, the server verifies whether it knows how to
parse first flights from the original version. If it does not, then it starts
incompatible version negotiation, see {{incompat-vn}}. If the server can parse
the first flight, it can either establish the connection using the original
version, or it MAY attempt compatible version negotiation, see {{compat-vn}}.

Note that it is possible for a server to have the ability to parse the first
flight of a given version without fully supporting it, in the sense that it
implements enough of the version's specification to parse first flight packets
but not enough to fully establish a connection using that version.


## Connections and Version Negotiation

QUIC connections are shared state between a client and a server {{INV}}. The
compatible version negotiation mechanism defined in this document (see
{{compat-vn}}) is performed as part of a single QUIC connection; that is, the
packets with the original version are part of the same connection as the packets
with the negotiated version.

In comparison, the incompatible version negotiation mechanism, which leverages
QUIC Version Negotiation packets (see {{incompat-vn}}) conceptually operates
across two QUIC connections: the connection attempt prior to receiving the
Version Negotiation packet is distinct from the connection with the incompatible
version that follows.


## Incompatible Version Negotiation {#incompat-vn}

The server starts incompatible version negotiation by sending a Version
Negotiation packet. This packet SHALL include each entry from the server's set
of Offered Versions (see {{server-fleet}}) in a Supported Version field. The
server MAY add reserved versions (as defined in {{Section 6.3 of QUIC}}) in
Supported Version fields.

Clients will ignore a Version Negotiation packet if it contains the original
version attempted by the client. The client also ignores a Version Negotiation
that contains incorrect connection ID fields; see {{Section 6 of INV}}.

Upon receiving the VN packet, the client will search for a version it supports
in the list provided by the server. If it doesn't find one, it aborts the
connection attempt. Otherwise, it selects a mutually supported version and sends
a new first flight with that version - we refer to this version as the
"negotiated version".

The new first flight will allow the endpoints to establish a connection using
the negotiated version. The handshake of the negotiated version will exchange
version information (see {{vers-info}}) required to ensure that VN was genuine,
i.e. that no attacker injected packets in order to influence the VN process, see
{{downgrade}}.


## Compatible Version Negotiation {#compat-vn}

When the server can parse the client's first flight using the original version,
it can extract the client's Version Information structure (see {{vers-info}}).
This contains the list of versions that the client knows its first flight is
compatible with.

If the server supports one of the client's compatible versions, and the server
also knows that the original version is compatible with this version, and the
client's first flight is compatible with this version, then the server converts
the client's first flight to that version and replies to the client as if it had
received the converted first flight. Note that this conversion process cannot
fail by definition of the first flight being compatible. The version used by the
server in its reply is refered to as the "negotiated version". The server MUST
NOT reply with a version that is not present in the client's compatible
versions, unless it is the original version.

If the server does not find a compatible version, it will use the original
version if it supports it, and if it doesn't then the server will perform
incompatible version negotiation instead, see {{incompat-vn}}.

Note that, after the first flight is converted to the negotiated version, the
handshake completes in the negotiated version. The entire handshake (including
the converted first flight) needs to conform to the rules of the negotiated
version. For instance, if the negotiated version requires that the 5-tuple
remain stable for the entire handshake (as QUIC version 1 does), then this
applies to the entire handshake, including the first flight.


# Version Information {#vers-info}

During the handshake, endpoints will exchange Version Information, which
consists of a chosen version and a list of other versions. Any version of QUIC
that supports this mechanism MUST provide a mechanism to exchange Version
Information in both directions during the handshake, such that this data is
authenticated.

In QUIC version 1, the Version Information is transmitted using a new transport
parameter, version_information. The contents of Version Information are shown
below (using the notation from the "Notational Conventions" section of {{QUIC}}):

~~~
Version Information {
  Chosen Version (32),
  Other Versions (32) ...,
}
~~~
{: #fig-vi-format title="Version Information Format"}

The content of each field is described below:

Chosen Version:

: The version that the sender has chosen to use for this connection. In most
cases, this field will be equal to the value of the Version field in the long
header that carries this data.

The contents of the Other Versions field depends on whether it is sent by the
client or by the server.

Client-Sent Other Versions:

: When sent by a client, the Other Versions field lists all the versions that
this first flight is compatible with, ordered by descending preference. Note
that the version in the Chosen Version field MUST be included in this list to
allow the client to communicate the chosen version's preference. Note that this
preference is only advisory, servers MAY choose to use their own preference
instead.

Server-Sent Other Versions:

: When sent by a server, the Other Versions field lists all the Fully-Deployed
Versions of this server deployment, see {{server-fleet}}. Note that the version
in the Chosen Version field is not necessarily included in this list, because
the server operator could be in the process of removing support for this
version. For the same reason, the Other Versions field MAY be empty.

Clients and servers MAY both include versions following the pattern 0x?a?a?a?a
in their Other Versions list. Those versions are reserved to exercise version
negotiation (see the Versions section of {{QUIC}}), and will never be selected
when choosing a version to use.


# Version Downgrade Prevention {#downgrade}

Clients MUST ignore any received Version Negotiation packets that contain the
version that they initially attempted. A client that makes a connection attempt
based on information received from a Version Negotiation packet MUST ignore any
Version Negotiation packets it receives in response to that connection attempt.

Both endpoints MUST parse their peer's Version Information during the handshake.
If parsing the Version Information failed (for example, if it is too short or if
its length is not divisible by four), then the endpoint MUST close the
connection; if the connection was using QUIC version 1, that connection closure
MUST use a transport error of type TRANSPORT_PARAMETER_ERROR.

Every QUIC version that supports version negotiation MUST define a method for
closing the connection with a version negotiation error. For QUIC version 1,
version negotiation errors are signaled using a transport error of type
VERSION_NEGOTIATION_ERROR; see {{iana-error}}.

If the Version Information was missing, the endpoints MAY complete the
handshake. However, if a client has reacted to a Version Negotiation packet and
the Version Information was missing, the client MUST close the connection with a
version negotiation error.

If the client received and acted on a Version Negotiation packet, the client
MUST validate the server's Other Versions field. The Other Versions field is
validated by confirming that the client would have attempted the same version
with knowledge of the versions the server supports. That is, the client would
have selected the same version if it received a Version Negotiation packet that
listed the versions in the server's Other Versions field, plus the negotiated
version. If the client would have selected a different version, the client MUST
close the connection with a version negotiation error. This connection closure
prevents an attacker from being able to use forged Version Negotiation packets
to force a version downgrade.

This validation of Other Versions is not sufficient to prevent downgrade.
Downgrade prevention also depends on the client ignoring Version Negotiation
packets that contain the original version; see {{incompat-vn}}.

After the process of version negotiation in this document completes, the version
in use for the connection is the version that the server sent in the Chosen
Version field of its Version Information. That remains true even if other
versions were used in the Version field of long headers at any point in the
lifetime of the connection; endpoints MUST NOT change the version that they
consider to be in use based on the Version field of long headers as that field
could be forged by attackers.

# Minimum support

Hosts can support this standard incrementally to simplify implementation, if
not all features of the design are desired.

{{INV}} and {{QUIC}} already describe how servers send Version Negotiation
packets in response to long header packets with unsupported version fields, even
if they support only one version. Doing so prevents long timeouts at the client,
but servers might not send the packet multiple times for the same connection, or
when under stress.

If a server supports two or more QUIC versions and sends Version Negotiation
packets, it MUST follow the requirements in {{vers-info}} and {downgrade}}.
Failure to comply would leave the connection vulnerable to downgrade attacks, or
cause connection failure, if the client initiates Incompatible Version
Negotiation ({{incompat-vn}}).

A server that supports two or more compatible QUIC versions MAY implement
Compatible version negotiation ({{compat-vn}}) if it prefers one version over
another.

A client that supports two or more versions, and chooses to respond to a Version
Negotiation packet with a connection attempt using one of the advertised
versions, MUST include a Version Information Transport Parameter and process the
server's transport parameter in accordance with {{downgrade}}.

If a client sends a Version Information Transport Parameter in a packet using a
version that is compatible with a version advertised in the transport parameter,
it MUST support compatible version negotiation.

Therefore, clients can only avoid support for compatible negotiation by either
(1) never attempting connection based on a Version Negotiation packet, and
electing not to include the transport parameter; or (2) never including a
compatible version in the trnasport parameter.

# Client Choice of Original Version

The client's first connection attempt SHOULD be made using the version that the
server is most likely to support. The client selects the version most likely to
be supported from the versions that are compatible with the client's most
preferred version. Without additional information this could mean selecting the
oldest version that the client supports.


# Interaction with Retry

QUIC version 1 features Retry packets, which the server can send to validate the
client's IP address before parsing the client's first flight. A server that
sends a Retry packet can do so before parsing the client's first flight. A
server that sends a Retry packet therefore might not have processed the client's
Version Information before doing so.

If a future document wishes to define compatibility between two versions that
support retry, that document MUST specify how version negotiation (both
compatible and incompatible) interacts with retry during a handshake that
requires both. For example, that could be accomplished by having the server send
a Retry packet first and validating the client's IP address before attempting
compatible version negotiation. In that scenario the Retry packet would be sent
using the original version.


# Interaction with 0-RTT

QUIC version 1 allows sending data from the client to the server during the
handshake, by using 0-RTT packets. If a future document wishes to define
compatibility between two versions that support 0-RTT, that document MUST
address the scenario where there are 0-RTT packets in the client's first flight.
For example, this could be accomplished by defining which transformations are
applied to 0-RTT packets. Alternatively, that document could specify that
compatible version negotiation causes 0-RTT data to be rejected by the server.


# Considerations for Future Versions

In order to facilitate the deployment of future versions of QUIC, designers of
future versions SHOULD attempt to design their new version such that commonly
deployed versions are compatible with it. For example, a successor to QUIC
version 1 may wish to design its transport parameters in a way that does not
preclude compatibility.


# Security Considerations

The security of this version negotiation mechanism relies on the authenticity of
the Version Information exchanged during the handshake. In QUIC version 1,
transport parameters are authenticated ensuring the security of this mechanism.
Negotiation between compatible versions will have the security of the weakest
common version.

The requirement that versions not be assumed compatible mitigates the
possibility of cross-protocol attacks, but more analysis is still needed here.


# IANA Considerations

## QUIC Transport Parameter

This document registers a new value in the QUIC Transport Parameter Registry
maintained at
[](https://www.iana.org/assignments/quic/quic.xhtml#quic-transport).

Value:

: 0xFF73DB

Parameter Name:

: version_information

Status:

: provisional

Specification:

: This document

When this document is approved, it will request permanent allocation of a
codepoint in the 0-63 range to replace the provisional codepoint described above.


## QUIC Transport Error Code {#iana-error}

This document registers a new value in the QUIC Transport Error Codes Registry
maintained at
[](https://www.iana.org/assignments/quic/quic.xhtml#quic-transport-error-codes).

Value:

: 0x53F8

Code:

: VERSION_NEGOTIATION_ERROR

Description:

: Error negotiating version

Status:

: provisional

Specification:

: This document

When this document is approved, it will request permanent allocation of a
codepoint in the 0-63 range to replace the provisional codepoint described above.


--- back

# Acknowledgments {#acknowledgments}
{:numbered="false"}

The authors would like to thank Nick Banks, Mike Bishop, Ryan Hamilton, Roberto
Peon, Anthony Rossi, and Martin Thomson for their input and contributions.
