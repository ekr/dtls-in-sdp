---
title: Piggybacked DTLS Handshakes in SDP
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




--- abstract
This document describes a mechanism for embedding DTLS handshake
messages in SDP descriptions. This technique allows implementations
to shave a full round-trip off of DTLS-SRTP session establishment,
while retaining compatibility with ordinary DTLS-SRTP endpoints.


--- middle

# Introduction

DTLS-SRTP {{!RFC5763}}{{!RFC5763}} uses
a DTLS {{!RFC6347}} handshake to establish keys which
are then used to key SRTP {{!RFC3711}}. The DTLS
negotiation is tied to the offer/answer {{!RFC3264}}
transaction via an "a=fingerprint" attribute
{{!RFC4572}} in the SDP {{!RFC4566}}. The
common message flow is shown below for DTLS 1.2.

This figure and the rest of this document adopt the following
assumptions about network behavior:

* ICE {{!RFC5245}} is in use but that both endpoints
  implement endpoint-independent filtering {{!RFC5389}} so
  that STUN checks succeed immediately.

* Signaling messages take the same time to be delivered
  as direct messages [this is generally false.]

Links to detailed diagrams with a more accurate vertical scale can
be found below each diagram.

~~~~
    Alice                 Signaling Service                 Bob
    -----------------------------------------------------------  
^   Offer + fingerprint  --------->
|                                 Offer + fingerprint -------->  ^  
|                                                                |
|                                 <------  Answer + fingerprint  |
|   <--------  Answer + fingerprint                              |
|   <------------------------------------------------- STUN-REQ  |
|   STUN-REQ ------------------------------------------------->  |
|   STUN-RESP------------------------------------------------->  |
|   <---------------------------------------------  ClientHello  |
|   <------------------------------------------------ STUN-RESP  |
4                                                                |
R   ServerHello                                                  |
T   ServerKeyExchange                                            |
T   Certificate                                                  3
|   CertificateRequest                                           R
|   ServerHelloDone  ----------------------------------------->  T
|                                                                T
|                                             ClientKeyExchange  |
|                                                   Certificate  |
|                                             CertificateVerify  |
|                                            [ChangeCipherSpec]  |
|   <------------------------------------------------  Finished  |
|                                                                |
|   [ChangeCipherSpec]                                           |
|   Finished ------------------------------------------------->  |
|   Media ---------------------------------------------------->  v
v   <------------------------ Media----------------------------
~~~~
{: #ordinary-dtls title="Standard DTLS-SRTP Negotiation"}

[Better picture](https://raw.githubusercontent.com/ekr/dtls-in-sdp/master/normal-12.png)

In this flow, the earliest that Alice can start sending media is
after receiving Bob's Finished and the earliest Bob can start
sending media is upon receiving Alice's Finished, and neither
side can send any DTLS messages until they have had a successful
STUN check. The result is that in the best case, Alice receives
media four round trips after sending the offer and Bob receives
media three round trips after receiving Alice's offer.

This document describes a technique for improving call setup time by
piggybacking the first round of DTLS messages on the signaling
messages. This reduces latency by a full round trip for both DTLS 1.2
and DTLS 1.3 handshakes, and for DTLS 1.3 {{!I-D.ietf-tls-tls13}}
allows the answerer to start sending media
immediately upon receiving the offer, or, if ICE is used, upon ICE
completion.


# Protocol Overview

The basic concept, as shown in {{piggybacked-dtls12}}, is for
Alice to send her ClientHello in her Offer and Bob to send
the server's first flight (ServerHello...ServerHelloDone for DTLS
1.2) in his Answer.

## DTLS 1.2

~~~~
    Alice                 Signaling Service                 Bob
    -----------------------------------------------------------
^   Offer
|    + fingerprint
|    + ClientHello      --------->
|                                 Offer
|                                  + fingerprint
|                                  + ClientHello  ------------>  ^
|                                                                |
|                                                        Answer  |
|                                                 + fingerprint  |
|                                                 + ServerHello  |
|                                           + ServerKeyExchange  |
|                                                 + Certificate  |
|                                          + CertificateRequest  |
|                                <------------  ServerHelloDone  |
3                                                                |
T                           Answer                               2
T                    + fingerprint                               R
T                    + ServerHello                               T
|              + ServerKeyExchange                               T
|                    + Certificate                               |
|             + CertificateRequest                               |
|   <------------  ServerHelloDone                               |
|   <------------------------------------------------  STUN-REQ  |
|   STUN-REQ ------------------------------------------------->  |
|   STUN-RESP------------------------------------------------->  |
|   <------------------------------------------------ STUN-RESP  |
|   ClientKeyExchange                                            |
|   Certificate                                                  |
|   CertificateVerify                                            |
|   [ChangeCipherSpec]                                           |
|   Finished ------------------------------------------------->  v
|   Media ---------------------------------------------------->
|                                            [ChangeCipherSpec]
|   <------------------------------------------------  Finished
v   <---------------------------------------------------  Media
~~~~
{: #piggybacked-dtls12 title="Piggybacked DTLS-SRTP Negotiation (TLS 1.2)"}

[Better picture](https://raw.githubusercontent.com/ekr/dtls-in-sdp/master/piggybacked-12.png)

Note that in this flow, the active/passive (DTLS client/server) roles
are reversed and Alice becomes the client. Because this is a basically
symmetrical transaction, this is not an issue.

It should be immediately apparent that this exchange shaves off a full
round trip from Bob's perspective (despite actually only shaving a
half a round trip from the number of messages). The reason is that Bob
does not need to wait for Alice's Finished to send but can piggyback
his data on his Finished.

This change also shaves off a round trip from Alice's perspective
because Alice can now safely perform TLS False Start
{{!I-D.ietf-tls-falsestart}} and send traffic prior to receiving Bob's
Finished message. When only fingerprints are carried in the handshake,
then extensions such as {{!RFC7301}} indicators and DTLS-SRTP
negotiation are not protected. However, in this case because those
indicators are carried in the hello messages which are now tied to the
signaling channel, they are authenticated via the same mechanisms
that authenticate the fingerprint.

Note: One could argue that under some conditions Bob could do
False Start in the ordinary handshake, but it's much harder to
analyze and even then it leaves Alice one round trip slower than
she would be with this optimization.

## DTLS 1.3

Figure {{piggybacked-dtls13}} shows the impact of this optimization
on DTLS 1.3. 


~~~~
    Alice                 Signaling Service                 Bob
    -----------------------------------------------------------
^   Offer
|    + fingerprint
|    + ClientHello      --------->
|                                 Offer
|                                  + fingerprint
|                                  + ClientHello  ------------>  ^
|                                                                |
|                                                        Answer  |
|                                                 + fingerprint  |
|                                                 + ServerHello  |
|                                          + CertificateRequest  |
|                                                 + Certificate  |
|                                           + CertificateVerify  |
|                                <-------------------  Finished  |
|                           Answer                               |
3                    + fingerprint                               |
R                    + ServerHello                               2
T             + CertificateRequest                               R
T                    + Certificate                               T
|              + CertificateVerify                               T
|   <-------------------  Finished                               |
|   <------------------------------------------------  STUN-REQ  |
|   STUN-REQ ------------------------------------------------->  |
|   STUN-RESP------------------------------------------------->  |
|   <------------------------------------------------ STUN-RESP  |
|                                                                |   
|   ClientKeyExchange                                            |
|   Certificate                                                  |
|   CertificateVerify                                            |
|   [ChangeCipherSpec]                                           |
|   Finished ------------------------------------------------->  |
|   Media ---------------------------------------------------->  v
|                                            [ChangeCipherSpec]
|   <------------------------------------------------  Finished
v   <---------------------------------------------------  Media
~~~~
{: #piggybacked-dtls13 title="Piggybacked DTLS-SRTP Negotiation (TLS 1.3)"}

[Better picture](https://raw.githubusercontent.com/ekr/dtls-in-sdp/master/piggybacked-13.png)

Alice cannot send any sooner than with DTLS 1.2
because sending at the point when she receives Bob's first
message is already optimal. It may be possible
for Bob to shave off yet another
round trip, however. As described in {{server-false-start}}.


# Attribute Definition

This document defines a new media-level SDP attribute, "a=dtls-message".
This message is used to contain DTLS messages. The syntax of this attribute
is:

~~~~
attribute               =/   dtls-message-attribute

dtls-message-attribute  =    "dtls-message" ":" role SP value

role                    =    "client" / "server"

value                   =    1*(ALPHA / DIGIT / "+" / "/" / "=" )
                             ; base64 encoded message
~~~~

An offeror which wishes to use the optimization defined in this document
shall send his ClientHello in the "a=dtls-message" attribute of its
initial offer with the role "client" and MUST use "a=setup:actpass". This allows the peer to
either:

* Reject the optimization, in which case it ignores the attribute.
* Accept the optimization, in which case it MUST use "a=setup:passive"
  and send its first flight (starting with ServerHello) and using
  the role "server" in its response. These messages are simply serialized
  end-to-end as they would be on the wire. It MAY also choose to 
  send its first flight separately in the media channel; DTLS implementations
  already handle retransmits properly.

The offerer MUST be able to detect whether an incoming DTLS message
is a ClientHello or a ServerHello and adapt accordingly.

In subsequent negotiations, implementations MUST maintain these
roles.


# Interactions

This optimization has a number of interactions with existing pieces of
protocol machinery.


## ICE

When ICE is in use, there is a race condition between the answerer's
ICE checks (at which point it will be able to send the first flight on
the media channel) and the answerer's Answer, which contains the first
flight. For this reason, we allow implementations to send the first
flight on both channels. However, as a practical matter it is
reasonably likely that when ICE is in use the Answer will arrive
first, for two reasons:

* The answerer consumes a full RTT doing a STUN check to verify
  the path to the offerer (even in the best case where the
  first STUN check succeeds). Thus, even if the path through
  the signaling server is twice as expensive as the direct path,
  there is a reasonable chance that the answer will arrive first.
  
* If the offerer is behind a NAT without endpoint-independent
filtering, the answerer's ICE checks will be discarded until the
offerer sends its own ICE checks, which it can only do upon receiving
the answer.

In this case, although a comparison of {{ordinary-dtls}} and
{{piggybacked-dtls12}} would show the ClientHello (in ordinary DTLS)
and the ServerHello (when piggybacked) as arriving at the same time,
in fact the ServerHello may arrive up to a full RTT first, but the
offerer can SEND its second flight immediately upon its STUN check
succeeding, which happens first, thus increasing the advantage of this technique.

## Forking

This technique does not interact very well with forking. Because each
ClientHello is only usable for one server, the system must somehow ensure
that only one of the forks takes up the piggybacked offers. The
easiest approach is for any intermediary which does a fork to strip
out the "a=dtls-message" attribute. An alternative would be to add
another attribute which could be stripped out (this might interact
better with RTCWEB Identity). Note that {{?RFC4474}} protects against
any SDP modifications, but I think at this point it's clear that that's
not practical.

## RTCWEB Identity

RTCWEB Identity assertions need to cover these DTLS messages.


# Examples

[we need examples.]

# Security Considerations

The security implications of this technique are described throughout
this document. 


# IANA Considerations

This specification defines the "dtls-message" SDP attribute per the
procedures of Section 8.2.4 of {{!RFC4566}}.  The required information
for the registration is included here:

~~~
   Contact Name:  Eric Rescorla (ekr@rftm.com)

   Attribute Name:  dtls-message

   Long Form:  dtls-message

   Type of Attribute:  session-level

   Charset Considerations:  This attribute is not subject to the charset
      attribute.

   Purpose:  This attribute carries piggybacked DTLS message.

   Appropriate Values:  This document
~~~

--- back

# Speculative: Server False-Start {#server-false-start}

WARNING: THE FOLLOWING SECTION HAS NOT RECEIVED ANY REAL SECURITY
REVIEW AND MAY BE A REALLY BAD IDEA.

It has been observed that as if Alice uses a fresh DH ephemeral, then Bob knows (because he can
trust the signaling service) that Alice's DH ephemeral corresponds
to Alice and can therefore encrypt under the joint DH shared
secret without waiting for Alice's CertificateVerify, as shown
in {{piggybacked-dtls13-false-start}}.

~~~~
    Alice                 Signaling Service                 Bob
    -----------------------------------------------------------
^   Offer
|    + fingerprint
|    + ClientHello      --------->
|                                 Offer
|                                  + fingerprint
|                                  + ClientHello  ------------>  ^
|                                                                |
|                                                        Answer  |
|                                                 + fingerprint  |
|                                                 + ServerHello  |
2                                          + CertificateRequest  |
R                                                 + Certificate  |
T                                           + CertificateVerify  |
T                                <-------------------  Finished  |
|                           Answer                               |
|                    + fingerprint                               |
|                    + ServerHello                               2
|             + CertificateRequest                               R
|                    + Certificate                               T
|              + CertificateVerify                               T
|   <-------------------  Finished                               |
|   <------------------------------------------------  STUN-REQ  |
|   STUN-REQ ------------------------------------------------->  |
|   STUN-RESP------------------------------------------------->  |
v   <---------------------------------------------------  Media  |
    <------------------------------------------------ STUN-RESP  |
                                                                 |   
    ClientKeyExchange                                            |
    Certificate                                                  |
    CertificateVerify                                            |
    [ChangeCipherSpec]                                           |
    Finished ------------------------------------------------->  |
    Media ---------------------------------------------------->  v
                                             [ChangeCipherSpec]
    <------------------------------------------------  Finished
~~~~
{: #piggybacked-dtls13-false-start title="Piggybacked DTLS-SRTP Negotiation (TLS 1.3 with false start)"}

[Better picture](https://raw.githubusercontent.com/ekr/dtls-in-sdp/master/piggybacked-13-falsestart.png)

This has demonstrably inferior security properties if Alice is
using a long-term key (for key continuity or fingerprint validation), because Bob has
not yet verified that Alice controls that key and does not even
know if Alice is using a fresh DH ephemeral, if implementations
decide to adopt this optimization, they must do something hacky like
Send data immediately but generate an error if the handshake,
including a signature, does not complete within some reasonable
period (a small number of measured round trips) [Just one reason
why this is a questionable technique.].



# Acknowledgements

Thanks to Cullen Jennings, Martin Thomson, and Justin Uberti for helpful
suggestions.




