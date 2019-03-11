---
title: Compatible Version Negotiation for QUIC
abbrev: QUIC VN
docname: draft-schinazi-quic-version-negotiation-latest
category: info

ipr: trust200902
area: General
workgroup: QUIC Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: D. Schinazi
    name: David Schinazi
    organization: Google
    email: dschinazi@google.com

    ins: E. Rescorla
    name: Eric Rescorla
    organization: Mozilla
    email: ekr@rtfm.com


normative:
  RFC2119:



--- abstract

QUIC does not provide a complete version negotiation mechanism but
instead only provides a way for the server to indicate that the
version the client offered is unacceptable. This document describes
a version negotiation mechanisnm that allows a client and server
to select from a set of QUIC versions which share a compatible
Initial format without incurring an extra round trip.



--- middle

# Introduction

QUIC {{!I-D.ietf-quic-transport}} does not provide a complete
version negotiation mechanism; the VN packet only allows the
server to indicate that the version the client offered is
unacceptable, but doesn't allow the client to safely make
use of that information. In principle the VN packet could be
part of a mechanism to allow two QUIC implementations to negotiate
between two totally disjoint versions of QUIC (at the cost of
an extra round trip). However, experience with negotiation of
previous IETF protocols indicates that this is probably not the
most common scenario:

1. Implementations do not generally want to incur an extra
   round trip to negotiate versions.
1. Most incremental versions are broadly similar to the the
   previous version, and so the version negotiation mechanism
   can be built on the assumption that the version advertisement
   and selection is common to the versions to be negotiated.

This specification describes a simple version negotiation mechanism
which exploits property (2) and can negotiate between the set
of "compatible" versions in a single round trip. Negotiation
between totally disjoint versions -- if it ever proves to be
necessary --  is left as a topic for future work.


# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.


# Version Negotiation Mechanism

The mechanism defined in this document is straightforward: the client supports a
set of ordered versions V_0 ... V_N. Its Initial packet is sent using the oldest
version that the client supports (V_0) and then lists all of the compatible
{{compatible-versions}} versions that the client supports in the
supported_versions field of its transport parameters {{supported-versions}}. The
server then selects its preferred version and responds with that version in all
of its future packets (except for Retry, as below). It also inserts the selected
version in the version field of its transport parameters.

The server MUST NOT select a version not offered by the client.  The client MUST
validate that the version in the server's packets is one of the versions that it
offered and that it matches the value in the server's transport parameters.

If the server sends a Retry, it MUST use the same version that the client
provided in its Initial. Version negotiation takes place after the retry cycle
is over.

In order for negotiation to complete successfully, the client's Initial packet
(and initial CRYPTO frames) MUST be interpretable by the server. This implies
that servers must retain the ability to process the Initial packet from older
versions as long as they are reasonably popular.  This is not generally an issue
in practice as long as the the overall structure of the protocol remains
similar.

If the server receives an Initial packet with a version it does not understand
this will cause a connection failure and the server SHOULD send a Version
Negotiation packet as defined in {{I-D.ietf-quic-transport}}

# Supported Versions Transport Parameter {#supported-versions}

This document adds a new transport parameter, SupportedVersions:

~~~~
struct {
      select (Handshake.msg_type) {
         case client_hello:
            QuicVersion supported_versions<4..2^8-4>;

         case server_hello:
            QuicVersion version;
      }
} SupportedVersions;
~~~~

The client's "supported_versions" parameter lists the versions it
supports in decreasing order of preference. The server's
parameter lists the version it has selected. If the client
does not send this transport parameter, the server MUST assume
that the client only supports the version it used for the
Initial packet and MUST NOT send its own parameter.


# Compatible Versions

Two versions of QUIC A and B are "compatible" if a version A Initial
can be used to negotiate version B and vice versa. The most common
scenario is a sequence of versions 1, 2, 3, etc. in which all the
Initial packets have the same basic structure but might include
specific extensions (especially inside the crypto handshake)
that are only meaningful in some subset of versions and are ignored
in others. Note that it is not possible to add new frame types
because QUIC requires that unrecognized frame types be treated
as an error. [[TODO: Citation needed]]


# Security Considerations



# IANA Considerations

IANA [SHALL assign/has assigned] the identifier TBD for the "supported_versions"
transport parameter.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
