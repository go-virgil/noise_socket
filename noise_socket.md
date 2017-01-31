---
title:      'The Noise Socket Protocol'
author:     'Alexey Ermishkin (scratch@virgilsecurity.com)'
revision:   '0'
date:       '2017-01-31'
---


1. Introduction
================

Noise Socket is a transport layer protocol that is much simplier than TLS,
easier to implement and does not require certificates, only raw public keys can
be used to establish secure connection.

It is useful in IoT, as TLS replacement in microservice architecture, messaging
and other cases where TLS looks overcomplicated.


It is based on the [Noise protocol framework](http://noiseprotocol.org) which
internaly uses only symmetric ciphers, hashes and DH to do secure handshakes.

2. Overview 
============ 
Noise Socket always uses [Noise_XX](http://noiseprotocol.org/noise.html#interactive-patterns) pattern
which can be extended to provide 0-RTT and other features. Noise_XX allows any
combination of authentications (client, server, mutual, none) by using null
public keys (i.e. sending a public key of zeros if you don't want to
authenticate).


3. Versioning and negotiation
---------------------------
For extensibility we'd like the ability for the client to offer
multiple Noise initial handshake messages, and for the server to
choose one:

First NoiseSocket message:
 - 2 bytes big-endian length of following data (N)
 - Repeat:
   - 1 byte version of following message (zero initially; sorted in
increasing order)
   - 2 bytes big-endian length of following message (DHLEN initially;
may be zero)
   - <Noise message>

Second NoiseSocket message:
 - 1 byte version (zero initially)
 - 2 bytes big-endian length of following message
 - <Noise message>

All subsequent messages:
 - 2 bytes big-endian length of following message
 - <Noise message>

The Noise "prologue" will be calculated as:
 - 1 byte indicating the number of versions offered in the first message
 - A list of bytes, one for each message.

For example, the initial prologue will be 0x0100 (1 version, version
0).  We don't include the entire NoiseSocket message so that the
prologue can be fixed to allow for encrypted initial messages, in the
future.  This means servers must decide which version to support based
purely on version numbers, not message contents.


EXAMPLES:

 * Suppose the client wants to offer a new protocol version (1).  It
simply appends the version 1 initial message to the version 0 initial
message, and sends them both in the initial NoiseSocket message.  The
server responds to the highest version it recognizes.

 * Suppose the client wants to offer three new ciphers (version 1-3)
but reuse the version 0 DH.  The client can send version indicators
for 1-3, each followed by a zero-length message, reusing the version 0
message's ephemeral public value.

 * Suppose we want to extend NoiseSocket to support 0-RTT via the
Noise Pipe design.  A version 1 could be used to indicate an IK
handshake attempt, reusing the version 0 message's ephemeral public
key, but containing the additional encrypted fields from the initial
IK message.  The server would respond to either version 0 (XX or XX
fallback) or version 1 (IK).


Naming
-------
A name like "NoiseSocket_25519_ChaChaPoly_BLAKE2b" would translate
into a Noise protocol name in the obvious way:

Noise_XX_25519_ChaChaPoly_BLAKE2b

For hybrid forward secrecy, adding the "hfs" transformation to get
"NoiseSockethfs_25519+NewHope..." is ugly.  Maybe we can improve
naming by saying that a name with a pair of public-key algorithms
defaults to the "hfs" transform, unless a different transform is
explicitly specified?  Then we get:

NoiseSocket_25519+NewHope_AESGCM_SHA256 ->

NoiseXX_25519+NewHope_AESGCM_SHA256