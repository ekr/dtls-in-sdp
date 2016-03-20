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
{{!RFC4916}} in the SDP {{!RFC4566}}. The
common message flow is shown below (for DTLS 1.2):


~~~~
    Alice                 Signaling Service                 Bob
    -----------------------------------------------------------  
^   Offer + fingerprint  --------->
|                                 Offer + fingerprint -------->  ^  
|                                                                |
|                                 <------  Answer + fingerprint  |
|   <--------  Answer + fingerprint                              |
|                                                                |
|   <---------------------------------------------  ClientHello  |
2                                                                |
R   ServerHello,                                                 |
T   ServerKeyExchange                                            |
T   Certificate                                                  2
|   CertificateRequest                                           R
|   ServerHelloDone  ----------------------------------------->  T
|                                                                T
|                                             ClientKeyExchange  |
|                                                   Certificate  |
|                                             CertificateVerify  |
|                                            [ChangeCipherSpec]  |
v   <------------------------------------------------  Finished  |
                                                                 |
    [ChangeCipherSpec]                                           |
    Finished ------------------------------------------------->  v

    <------------------------ SRTP --------------------------->
~~~~
{: #ordinary-dtls title="Standard DTLS-SRTP Negotiation"}

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
target="I-D.ietf-tls-tls13}} allows the answerer to start sending media
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
1                                                 + fingerprint  |
R                                                 + ServerHello  |
T                                           + ServerKeyExchange  |
T                                                 + Certificate  |
|                                          + CertificateRequest  |
|                                <------------  ServerHelloDone  |
|                                                                |
|                           Answer                               1
|                    + fingerprint                               R
|                    + ServerHello                               T
|              + ServerKeyExchange                               T
|                    + Certificate                               |
|             + CertificateRequest                               |
v   <------------  ServerHelloDone                               |
                                                                 |
    ClientKeyExchange                                            |
    Certificate                                                  |
    CertificateVerify                                            |
    [ChangeCipherSpec]                                           |
    Finished ------------------------------------------------->  v
    
                                             [ChangeCipherSpec]
    <------------------------------------------------  Finished
    
    <------------------------ SRTP --------------------------->
~~~~
{: #piggybacked-dtls12 title="Piggybacked DTLS-SRTP Negotiation (TLS 1.2)"}

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
that authenticate the fingerprint (see {{oob-fingerprint}} for
more details.

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
|                                                                v
|                                                        Answer  
|                                                 + fingerprint  
1                                                 + ServerHello  
R                                          + CertificateRequest  
T                                                 + Certificate  
T                                           + CertificateVerify   
|                                <-------------------  Finished  
|                           Answer  
|                    + fingerprint  
|                    + ServerHello  
|             + CertificateRequest  
|                    + Certificate  
|              + CertificateVerify   
v   <-------------------  Finished  
                                                                 
    ClientKeyExchange                                            
    Certificate                                                  
    CertificateVerify                                            
    [ChangeCipherSpec]                                           
    Finished ------------------------------------------------->  
    
                                             [ChangeCipherSpec]
    <------------------------------------------------  Finished
    
    <------------------------ SRTP --------------------------->
~~~~
{: #piggybacked-dtls13 title="Piggybacked DTLS-SRTP Negotiation (TLS 1.3)"}
     
Alice cannot send any sooner than with TLS 1.2
because sending at the point when she receives Bob's first
message is already optimal. However, Bob can shave off yet another
round trip because he can send immediately upon receiving Alice's
first message. The reason for this is that as long as Alice
uses a fresh DH ephemeral, then Bob knows (because he can
trust the signaling service) that Alice's DH ephemeral corresponds
to Alice and can therefore encrypt under the joint DH shared
secret without waiting for Alice's CertificateVerify.
Arguably, we need not even do DTLS client authentication in this
case, but it is a good idea for protocol consistency and for
linkage to the long-term credential (see {{oob-fingerprint}}).


# Attribute Definition



# Interactions

## ICE

## Forking

## RTCWEB Identity

## Out-of-Band Fingerprint Validation {#oob-fingerprint}








