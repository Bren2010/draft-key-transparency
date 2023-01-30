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
Transparency  {{?I-D.ietf-trans-rfc6962-bis}} is that KT includes an efficient
protocol to search the log for entries related to a specific participant. This
means that users don't need to download the entire log, which may be
substantial, to find all entries that are relevant to them. It also means that
KT can better preserve user privacy by only showing entries of the log to
participants that genuinely need to see them.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Protocol Overview

From a networking perspective, KT follows a client-server architecture with a
central *Transparency Log*, acting as a server, which holds the authoritative
copy of all information and exposes endpoints that allow clients to query or
modify stored data. Clients coordinate with each other through the server by
uploading their own public keys and downloading the public keys of other
clients. Clients are expected to maintain relatively little state, limited only
to what is required to interact with the log and ensure that it is behaving
honestly.

From an application perspective, KT works as a versioned key-value database.
Clients insert key-value pairs into the database where, for example, the key is
their username and the value is their public key. Clients can update a key by
inserting a new version with new data. They can also lookup the most recent
version of a key or any past version. From this point forward, "key" will refer
to a lookup key in a key-value database and "public key" or "private key" will
be specified if otherwise.

Note that while this document provides specific protocol messages and the way
that those messages would be encoded, it does not require the use of a specific
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
can be utilized effectively when/if they're available. <!-- TODO: Link later section -->

## Operational Modes

TODO

## Basic Protocols

The protocols that can be executed by a client are as follows:

1. **Search:** Looks up a specific key in the most recent version of the log.
   Clients may request either a specific version of the key, or the most recent
   version available. If the key/version exists, the server returns the
   corresponding value and a proof of inclusion. If the key/version does
   not exist, the server returns a proof of non-inclusion instead.
2. **Update:** Adds a new key-value pair to the log and returns a proof of
   inclusion. Note that this means insertions are completed immediately and are
   not subject to a delay.
3. **Monitor:** While Search and Update are run by the client as-needed,
   monitoring is done in the background on a recurring basis. It both checks
   that the log is continuing to behave honestly and that no changes have been
   made to keys owned by the client without the client's knowledge.

## Security Guarantees

A client that executes a Search or Update protocol correctly (and does any
required monitoring afterwards) receives a guarantee that the Transparency Log
operator also executed the protocol correctly and in a way that's globally
consistent with what it has shown all other clients. That is, when a client
searches for a key, they're guaranteed that the result they receive represents
the same result that any other client searching for the same key would've seen.
When a client updates a key, they're guaranteed that other clients will see the
update the next time they search for the key. <!-- subject to caching? -->

If the Transparency Log operator does not execute the protocol correctly, then
either:

1. The client will detect the error immediately and reject the result, or
2. The client will permanently enter an invalid state.

Depending on the exact reason that the client enters an invalid state, it will
either be detected by background monitoring or the next time that out-of-band
communication is available. Importantly, this means that clients must stay
online for some fixed amount of time after entering an invalid state for it to
be successfully detected. <!-- need oob communication with someone not attacked? -->

The exact caveats of the above guarantee depend naturally on the security of
underlying cryptographic primitives, but also the operational mode that the
Transparency Log relies on:

- Contact Monitoring requires an assumption that the client that owns a key and
  all clients that look up the key do the required monitoring afterwards.
- Third-Party Management and Third-Party Auditing require an assumption that the
  Transparency Log operator and the third-party manager/auditor do not collude
  to trick clients into accepting malicious changes.
  <!-- write down why collusion-resistant KT is better than just having two operators stay in sync? -->

<!-- TODO: Once the whole protocol is written, ensure this is as precise as possible. -->
<!-- TODO: In Security Considerations, calculate how long you need to stay online for Contact Monitoring to detect an attack -->


# Tree Construction

KT relies on two combined hash tree structures: log trees and prefix trees. This
section describes the operation of both types of trees at a high-level and the
way that they're combined. More precise algorithms for computing the
intermediate and root values of the trees will be given in a later section.
<!-- TODO: Link later section -->

All types of trees consist of *nodes* which have a byte string as their *value*.
A node is either a *leaf* if it has no children, or a *parent* if it has either
a *left child* or a *right child*. A node is the *root* of a tree if it has no
parents, and an *intermediate* if it has both children and parents. Nodes are
*siblings* if they share the same parent.

The *descendants* of a node are that node, its children, and the descendants of
its children. A *subtree* of a tree is the tree given by the descendants of any
node, called the *head* of the subtree.

The *direct path* of a root node is the empty list, and of any other node is the
concatenation of that node's parent along with the parent's direct path. The
*copath* of a node is the node's sibling concatenated with the list of siblings
of all the nodes in its direct path, excluding the root.

## Log Tree

Log trees are used for storing information in the chronological order that it
was added and are constructed as *left-balanced* binary trees.

A binary tree is *balanced* if its size is a power of two and for any parent
node in the tree, its left and right subtrees have the same size. A binary tree
is *left-balanced* if for every parent, either the parent is balanced, or the
left subtree of that parent is the largest balanced subtree that could be
constructed from the leaves present in the parent's own subtree. Given a list of
`n` items, there is a unique left-balanced binary tree structure with these
elements as leaves. Note also that in this tree, every parent always has both a
left and right child.

Log trees initially consist of a single leaf node. New leaves are added to the
right-most edge of the tree along with a single parent node, to construct the
left-balanced binary tree with `n+1` leaves.

While the value of a leaf is arbitrary, the value of a parent node is always the
hash of the combined values of its left and right children.

Log trees are special in that they can provide both *inclusion proofs*, which
demonstrate that a leaf is included in a given tree, and *consistency proofs*,
which demonstrate that a new version of a log is an extension of a past version
of the log.

An inclusion proof is given by providing the copath values of a leaf. The proof
is verified by hashing together the leaf with the copath values and checking
that the result equals the root value of the log. Consistency proofs are a more
general version of the same idea. With a consistency proof, the prover provides
the minimum set of intermediate node values from the current tree that allows
the verifier to compute both the old root value and the current root value.

## Prefix Tree

Prefix trees are used for storing key-value pairs while preserving the ability
to efficiently look up a value by its corresponding key.

Each leaf node in a prefix tree contains the encoded data of a key-value pair.
Each parent node represents some specific prefix which all keys in the subtree
headed by that node have in common. The subtree headed by a parent's left child
contains all keys that share its prefix followed by an additional 0 bit, while
the subtree headed by a parent's right child contains all keys that share its
prefix followed by an additional 1 bit.

The root node, in particular, represents the empty string as a prefix. The
root's left child contains all keys that begin with a 0 bit, while the right
child contains all keys that begin with a 1 bit.

A prefix tree can be searched by starting at the root node, and moving to the
left child if the first bit of a search key is 0, or the right child if the
first bit is 1. This is then repeated for the second bit, third bit, and so on
until the search either terminates at a leaf node (which may or may not be for
the desired key), or a parent node that lacks the desired child.

New key-value pairs are added to the tree by searching it according to this
process. If the search terminates at a parent without a left or right child, a
new leaf is simply added as the parent's missing child. If the search terminates
at a leaf for the wrong key, one or more intermediate nodes are added until the
new leaf and the old leaf would no longer reside in the same place. That is,
until we reach the first bit that differs between the new key and the old key.

The value of a leaf node is the encoded key-value pair, while the value of a
parent node is the hash of the combined values of its left and right children
(or a stand-in value when one of the children doesn't exist).

## Combined Tree

Log trees are desirable because they can provide efficient consistency proofs to
ensure verifiers that nothing has been removed from a log that was present in a
previous version. However, log trees can't be efficiently searched without
downloading the entire log. Prefix trees are extremely efficient to search, and
can provide inclusion proofs to convince verifiers that the returned search
results are correct. However, it's not possible to efficiently prove that a new
version of a prefix tree contains the same data as a previous version (with only
new keys added).

In the combined tree structure, a log tree maintains a record of updates to
key-value pairs while a prefix tree maintains a map from each key to the number
of times it's been updated. Importantly, the root value of the prefix tree after
applying a given update is then stored in the log tree alongside the record of
the update. With some caveats, this combined structure supports both efficient
consistency proofs and can be efficiently searched.


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
