---
title: DTLS Handshake in SDP
abbrev: DTLS in SDP
docname: draft-rescorla-dtls-in-sdp
category: info

ipr: trust200902
area: General
workgroup: RTCWEB WG
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
       ins: E. Rescorla
       name: Eric Rescorla
       organization: RTFM, Inc.
       email: ekr@rtfm.com

normative:
  RFC2119:
  RFC3264:
  RFC3711:
  RFC4566:
  RFC4916:
  RFC5763:
  RFC5764:
  RFC6347:
  I-D.ietf-tls-tls13:

informative:




--- abstract
This document describes a mechanism for embedding DTLS handshake
messages in SDP descriptions. This technique allows implementations
to shave a full round-trip off of DTLS-SRTP session establishment,
while retaining compatibility with ordinary DTLS-SRTP endpoints.


--- middle

Introduction        {#problems}
============
DTLS-SRTP <xref target="RFC5763"/><xref target="RFC5763"/> uses
a DTLS <xref target="RFC6347"/> handshake to establish keys which
are then used to key SRTP <xref target="RFC3711"/>. The DTLS
negotiation is tied to the offer/answer <xref target="RFC3264"/>
transaction via an "a=fingerprint" attribute
<xref target="RFC4916"/> in the SDP <xref target="RFC4566"/>. The
common message flow is shown below (for DTLS 1.2):


~~~~
    Alice                 Signaling Service                 Bob
    -----------------------------------------------------------
    Offer + fingerprint  --------->
                                  Offer + fingerprint -------->
    
                                  <------  Answer + fingerprint
    <--------  Answer + fingerprint
    
    <---------------------------------------------  ClientHello
    
    ServerHello,
    ServerKeyExchange
    Certificate
    CertificateRequest
    ServerHelloDone  ----------------------------------------->
    
                                              ClientKeyExchange
                                                    Certificate
                                              CertificateVerify
                                             [ChangeCipherSpec]
    <------------------------------------------------  Finished
    
    [ChangeCipherSpec]
    Finished ------------------------------------------------->

    <------------------------ SRTP --------------------------->
~~~~

In this flow, the earliest that Alice can start sending media is
after receiving Bob's Finished, after two round trips (if we assume
that the signaling messages travel as fast as media messages,
which is generally not true). Similarly, Bob can start sending
two round trips after receiving Alice's offer, upon receiving
Alice's Finished. The situation is even worse with ICE, leading
to long call setup times.

This document describes a technique for improving call setup time by
piggybacking the first round of DTLS messages on the signaling
messages. This reduces latency by a full round trip for both DTLS 1.2
and DTLS 1.3 handshakes, and for DTLS 1.3 <xref
target="I-D.ietf-tls-tls13"/> allows the answerer to start sending media
immediately upon receiving the offer, or, if ICE is used, upon ICE
completion.







