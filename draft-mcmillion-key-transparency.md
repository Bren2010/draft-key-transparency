---
title: "Key Transparency"
category: info

docname: draft-mcmillion-key-transparency-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: SEC
workgroup: KT Working Group
keyword:
 - end-to-end encryption
 - append-only log
# venue:
#   group: WG
#   type: Working Group
#   mail: WG@example.com
#   arch: https://example.com/WG
#   github: USER/REPO
#   latest: https://example.com/LATEST

author:
 -
    fullname: Brendan McMillion
    email: brendanmcmillion@gmail.com

normative:

informative:
  RFC6962: DOI.10.17487/RFC6962


--- abstract

While there are several established protocols for end-to-end encryption,
relatively little attention has been given to securely distributing the end-user
public keys for encryption. As such, encryption protocols are often still
vulnerable to eavesdropping by active attackers. Key Transparency is a protocol
for distributing sensitive cryptographic information, such as public keys, in a
way that reliably either prevents interference or detects that it occured in a
timely manner. More generally though, it can also be applied to ensure that a
group of users agree on a shared value or to keep tamper-evident logs of
security-critical events.

--- middle

# Introduction

Before any information can be exchanged in an end-to-end encrypted system, two
things must happen. First, participants in the system must provide any public
keys they intend to use for encryption to the service operator. Second, these
public keys must be somehow distributed to any participants that will rely on
them for decryption.

Typically this is done simply by having users upload their public keys to a
directory and allowing other users to download from the directory as-needed.
With this approach, the service operator is trusted not to manipulate the
directory by inserting malicious public keys. As such, any encryption protocol
can really only protect users against passive eavesdropping on their messages.

However most messaging systems are designed such that all messages exchanged
flow through the service operator's servers, which means that it's extremely
easy for an operator to launch an active attack. That is, the service operator
can insert public keys into the directory that they know the private key for,
attach those public keys to a user's account without the user's knowledge, and
then inject these keys into active conversations with that user to receive
plaintext data.

Key Transparency (KT) solves this problem by requiring the service operator to
store user public keys in a cryptographically-protected append-only log. Any
malicious entries added to such a log will generally be equally visible to all
users, in which case a user can trivially detect that they're being impersonated
by viewing the public keys attached to their account. However, if the service
operator attempts to conceal some entries of the log from some users but not
others, this creates a "forked view" of the log which is permanent and
imminently detectable by any out-of-band communication.

The critical improvement of KT over related protocols like Certificate
Transparency {{RFC6962}} is that KT includes an efficient protocol to search the
log for entries related to a specific participant. This means that users don't
need to download the entire log, which may be substantial, to find all entries
that are relevant to them. It also means that KT can better preserve user
privacy by only showing entries of the log to participants that genuinely need
to see them.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# System Model

From a networking perspective, the protocol in this document follows a
client-server architecture with a central *Transparency Log*, acting as a
server, which holds the authoritative copy of all information and exposes
endpoints that allow clients to query or modify stored data. Clients coordinate
with each other through the server by uploading their own public keys and
downloading the public keys of other clients. Clients are expected to maintain
relatively little state, limited only to what is required to interact with the
log and ensure that it is behaving honestly.

From an application perspective, KT works as a versioned key-value database.
Clients insert key-value pairs into the database where, for example, the key is
their username and the value is their public key. Clients can update a key by
inserting a new version with new data. They can also lookup the most recent
version of a key or any past version. From this point forward, "key" will refer
to a lookup key in a key-value database and "public key" or "private key" will
be specified if otherwise.

Note that while this document provides specific protocol messages and the way
that those messages should be encoded, it does not require the use of a specific
transport protocol. This is intended to allow applications to layer KT on top of
whatever transport protocol their application already uses. In particular, this
allows applications to continue relying on their existing access control system.

Applications may enforce arbitrary access control rules on top of KT such as
requiring a user to be logged in to make KT requests, only allowing a user to
lookup the keys of another user if they're "friends", or simply applying a rate
limit. Applications SHOULD prevent users from modifying keys that they don't
own. The exact mechanism for rejecting requests, and possibly explaining the
reason for rejection, is left to the application.

Finally, this document does not assume that clients can reliably communicate
with each other out-of-band (that is, away from any interference by the
Transparency Log operator), or communicate with the Transparency Log
anonymously. However, later sections will give guidance on how these channels
can be utilized effectively when/if they're available.

## Protocols

## Operational Modes

## Security Guarantees

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
