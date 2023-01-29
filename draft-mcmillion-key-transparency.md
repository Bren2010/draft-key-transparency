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

## Operating Model



# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
