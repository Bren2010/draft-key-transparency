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
  RFC2104: DOI.10.17487/RFC2104
  RFC6962: DOI.10.17487/RFC6962

informative:
  Merkle2:
    target: https://eprint.iacr.org/2021/453
    title: "Merkle^2: A Low-Latency Transparency Log System"
    date: 2021-04-08
    author:
      - name: Yuncong Hu
      - name: Kian Hooshmand
      - name: Harika Kalidhindi
      - name: Seung Jin Yang
      - name: Raluca Ada Popa

--- abstract

While there are several established protocols for end-to-end encryption,
relatively little attention has been given to securely distributing the end-user
public keys for such encryption. As a result, these protocols are often still
vulnerable to eavesdropping by active attackers. Key Transparency is a protocol
for distributing sensitive cryptographic information, such as public keys, in a
way that reliably either prevents interference or detects that it occured in a
timely manner. In addition to distributing public keys, it can also be applied
to ensure that a group of users agree on a shared value or to keep
tamper-evident logs of security-critical events.

--- middle

# Introduction

Before any information can be exchanged in an end-to-end encrypted system, two
things must happen. First, participants in the system must provide to the
service operator any public keys they wish to use to receive messages. Second,
the service operator must somehow distribute these public keys to any
participants that wish to send messages to those users.

Typically this is done by having users upload their public keys to a simple
directory where other users can download them as necessary. With this approach,
the service operator is trusted not to manipulate the directory by inserting
malicious public keys, which means the underlying encryption protocol can only
protect users against passive eavesdropping on their messages.

However most messaging systems are designed such that all messages exchanged
flow through the service operator's servers, so it's extremely easy for an
operator to launch an active attack. That is, the service operator can insert
public keys into the directory that they know the private key for, attach those
public keys to a user's account without the user's knowledge, and then inject
these keys into active conversations with that user to receive plaintext data.

Key Transparency (KT) solves this problem by requiring the service operator to
store user public keys in a cryptographically-protected append-only log. Any
malicious entries added to such a log will generally be equally visible to all
users, in which case a user can trivially detect that they're being impersonated
by viewing the public keys attached to their account. However, if the service
operator attempts to conceal some entries of the log from some users but not
others, this creates a "forked view" which is permanent and easily detectable
with out-of-band communication.

The critical improvement of KT over related protocols like Certificate
Transparency  {{RFC6962}} is that KT includes an efficient
protocol to search the log for entries related to a specific participant. This
means users don't need to download the entire log, which may be substantial, to
find all entries that are relevant to them. It also means that KT can better
preserve user privacy by only showing entries of the log to participants that
genuinely need to see them.


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

While this document uses the TLS presentation language {{!RFC8446}} to describe
the structure of protocol messages, it does not require the use of a specific
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
can be utilized effectively when or if they're available. <!-- TODO: Link later section -->

## Basic Operations

The operations that can be executed by a client are as follows:

1. **Search:** Looks up a specific key in the most recent version of the log.
   Clients may request either a specific version of the key, or the most recent
   version available. If the key/version exists, the server returns the
   corresponding value and a proof of inclusion. If the key/version does
   not exist, the server returns a proof of non-inclusion instead.
2. **Update:** Adds a new key-value pair to the log and returns a proof of
   inclusion. Note that this means insertions are completed immediately and are
   not subject to a delay.
3. **Monitor:** While Search and Update are run by the client as necessary,
   monitoring is done in the background on a recurring basis. It both checks
   that the log is continuing to behave honestly and that no changes have been
   made to keys owned by the client without the client's knowledge.

## Deployment Modes

In the interest of satisfying the widest range of use-cases possible, three
different modes for deploying a Transparency Log are described in this document.
Each mode has slightly different requirements and efficiency considerations for
both the service operator and the end-user.

**Third-party Management** and **Third-party Auditing** are two deployment
modes that require the service operator to delegate part of the operation of the
Transparency Log to a third-party. Users are able to run more efficiently
as long as they can assume that the service operator and the third-party won't
collude to trick them into accepting malicious results.

With both third-party modes, all requests from end-users are initially
routed to the service operator and the service operator coordinates with the
third-party themself. End-users never contact the third-party directly, however
they will need a signature public key from the third-party to verify the
third-party's assertions.

With Third-party Management, the third-party performs the majority of the work
of actually storing and operating the log, and the service operator only needs
to sign new entries as they're added. With Third-party Auditing, the service
operator performs the majority of the work of storing and operating the log, and
obtains signatures from a lightweight third-party auditor at regular intervals
asserting that the service operator has been constructing the tree correctly.

**Contact Monitoring**, on the other hand, supports a single-party deployment
with no third-party. The tradeoff is that the background monitoring protocol
requires a number of requests that's proportional to the number of keys a user
has looked up in the past. As such, it's less suited to use-cases where users
look up a large number of ephemeral keys, but would work ideally in a use-case
where users look up a small number of keys repeatedly (for example, the keys of
regular contacts).

The deployment mode of a Transparency Log is chosen when the log is first
created and isn't able to be changed over the log's lifetime. This makes it
important for operators to carefully consider the best long-term approach based
on the specifics of their application.

## Security Guarantees

A client that executes a Search or Update operation correctly (and does any
required monitoring afterwards) receives a guarantee that the Transparency Log
operator also executed the operation correctly and in a way that's globally
consistent with what it has shown all other clients. That is, when a client
searches for a key, they're guaranteed that the result they receive represents
the same result that any other client searching for the same key would've seen.
When a client updates a key, they're guaranteed that other clients will see the
update the next time they search for the key. <!-- subject to caching? -->

If the Transparency Log operator does not execute an operation correctly, then
either:

1. The client will detect the error immediately and reject the result of an operation, or
2. The client will permanently enter an invalid state.

Depending on the exact reason that the client enters an invalid state, it will
either be detected by background monitoring or the next time that out-of-band
communication is available. Importantly, this means that clients must stay
online for some fixed amount of time after entering an invalid state for it to
be successfully detected. <!-- need oob communication with someone not attacked? -->

The exact caveats of the above guarantee depend naturally on the security of
underlying cryptographic primitives, but also the deployment mode that the
Transparency Log relies on:

- Third-Party Management and Third-Party Auditing require an assumption that the
  Transparency Log operator and the third-party manager/auditor do not collude
  to trick clients into accepting malicious results.
- Contact Monitoring requires an assumption that the client that owns a key and
  all clients that look up the key do the necessary monitoring afterwards.
  <!-- write down why collusion-resistant KT is better than just having two operators stay in sync? -->

<!-- TODO: Once the whole protocol is written, ensure this is as precise as possible. -->
<!-- TODO: In Security Considerations, calculate how long you need to stay online for Contact Monitoring to detect an attack -->


# Tree Construction

KT relies on two combined hash tree structures: log trees and prefix trees. This
section describes the operation of both at a high-level and the way that they're
combined. More precise algorithms for computing the intermediate and root values
of the trees will be given in a later section.
<!-- TODO: Link later section -->

Both types of trees consist of *nodes* which have a byte string as their *value*.
A node is either a *leaf* if it has no children, or a *parent* if it has either
a *left child* or a *right child*. A node is the *root* of a tree if it has no
parents, and an *intermediate* if it has both children and parents. Nodes are
*siblings* if they share the same parent.

The *descendants* of a node are that node, its children, and the descendants of
its children. A *subtree* of a tree is the tree given by the descendants of a
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
elements as leaves. Note also that every parent always has both a left and right
child.

Log trees initially consist of a single leaf node. New leaves are added to the
right-most edge of the tree along with a single parent node, to construct the
left-balanced binary tree with `n+1` leaves.

While leaves contain arbitrary data, the value of a parent node is always the
hash of the combined values of its left and right children.

Log trees are special in that they can provide both *inclusion proofs*, which
demonstrate that a leaf is included in a log, and *consistency proofs*, which
demonstrate that a new version of a log is an extension of a past version of the
log.

An inclusion proof is given by providing the copath values of a leaf. The proof
is verified by hashing together the leaf with the copath values and checking
that the result equals the root value of the log. Consistency proofs are a more
general version of the same idea. With a consistency proof, the prover provides
the minimum set of intermediate node values from the current tree that allows
the verifier to compute both the old root value and the current root value. An
algorithm for this is given in section 2.1.2 of {{RFC6962}}.

## Prefix Tree

Prefix trees are used for storing key-value pairs while preserving the ability
to efficiently look up a value by its corresponding key.

Each leaf node in a prefix tree represents a specific key-value pair, while each
parent node represents some prefix which all keys in the subtree headed by that
node have in common. The subtree headed by a parent's left child contains all
keys that share its prefix followed by an additional 0 bit, while the subtree
headed by a parent's right child contains all keys that share its prefix
followed by an additional 1 bit.

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
assure verifiers that nothing has been removed from a log that was present in a
previous version. However, log trees can't be efficiently searched without
downloading the entire log. Prefix trees are efficient to search and can provide
inclusion proofs to convince verifiers that the returned search results are
correct. However, it's not possible to efficiently prove that a new version of a
prefix tree contains the same data as a previous version with only new keys
added.

In the combined tree structure, which is based on {{Merkle2}}, a log tree
maintains a record of updates to key-value pairs while a prefix tree maintains a
map from each key to a counter with the number of times it's been updated. Importantly, the
root value of the prefix tree after adding the new key or increasing the counter
of an existing key, is stored in the log tree alongside the record of the
update. With some caveats, this combined structure supports both efficient
consistency proofs and can be efficiently searched.

To search the combined structure, users do a binary search for the first log
entry where looking up the search key in the prefix tree at that entry yields
the desired counter. As such, the entry that a user arrives at through binary
search contains the update with the key-value pair that they're looking for,
even though the log itself is not sorted.

Binary search also ensures that all users will check the same or similar entries
when searching for the same key, which is necessary for the efficient auditing
of a Transparency Log. To maximize this effect, users start their binary search
at the entry whose index is the largest power of two less than the size of the
log, and move left or right by consecutively smaller powers of two.

So for example in a log with 70 entries, instead of starting a search at the
"middle" with entry 35, users would start at entry 64. If the next step in the
search is to move right, instead of moving to the middle of entries 64 and 70,
which would be entry 67, users would move 4 steps (the largest power of two
possible) to the right to entry 68. As more entries are added to the log, users
will consistently revisit entries 64 and 68, while they may never revisit
entries 35 or 67 after even a single new entry is added to the log.

While users searching for a specific version of a key jump right into a binary
search for the entry with that counter, other users may instead wish to search
for the "most recent" version of a key. That is, the key with the highest
counter possible. Users looking up the most recent version of a key start by
fetching the **frontier** of the log, which they use to determine what the
highest counter for a key is.

The frontier of a log consists of the entry whose index is the largest power of
two less than the size of the log, followed by the entry the largest power of
two possible to the right, repeated until the last entry of the log is reached.
So for example in a log with 70 entries, the frontier consists of entries 64,
68, and 69 (with 69 being the last entry).

# Preserving Privacy

In addition to being more convenient for many use-cases than similar
transparency protocols, KT is also better at preserving the privacy of a
Transparency Log's contents. This is important because in many practical
applications of KT, service operators expect to be able to control when
sensitive information is revealed. In particular, an operator can often only
reveal that a user is a member of their service to that user's friends or
contacts. Operators may also wish to conceal when individual users perform a
given task like rotate their public key or add a new device to their account, or
even conceal the exact number of users their application has overall.

Applications are primarily able to manage the privacy of their data in KT by
enforcing access control policies on the basic operations performed by clients,
as discussed in {{protocol-overview}}. However, the proofs of inclusion and
non-inclusion given by a Transparency Log can indirectly leak information about
other entries and lookup keys.

When users search for a key with the binary search algorithm described in
{{combined-tree}}, they necessarily see the values of several leaves while
conducting their search that they may not be authorized to view the contents of.
However, log entries generally don't need to be inspected except as specifically
allowed by the service.

The privacy of log entries is maintained by storing only a cryptographic
commitment to the serialized, updated key-value pair in the leaf of the log tree
instead of the update itself. At the end of a successful search, the service
operator provides the committed update along with the commitment opening, which
allows the user to verify that the commitment in the log tree really does
correspond to the provided update. By logging commitments instead of plaintext
updates, users learn no information about an entry's contents unless the service
operator explicitly provides the commitment opening.

Beyond the log tree, the second potential source of privacy leaks is the prefix
tree. When receiving proofs of inclusion and non-inclusion from the prefix tree,
users also receive indirect information about what other valid lookup keys
exist. To prevent this, all lookup keys are processed through a Verifiable
Random Function, or VRF {{!I-D.irtf-cfrg-vrf}}.

A VRF deterministically maps each key to a fixed-length pseudorandom value. The
VRF can only be executed by the service operator, who holds a private key.
But critically, VRFs can still provide a proof that an input-output pair is valid,
which users verify with a public key. When a user requests to search for or
update a key, the service operator first executes its VRF on the input key to
obtain the output key that will actually be looked up or stored in the prefix
tree. The service operator then provides the output key, along with a proof that
the output key is correct, in its response to the user.

The pseudorandom output of VRFs means that even if a user indirectly observes
that a search key exists in the prefix tree, they can't immediately learn which
user the search key identifies. The inability of users to execute the VRF
themselves also prevents offline "password cracking" approaches, where an
attacker tries all possibilities in a low entropy space (like the set of phone
numbers) to find the input that produces a given search key.


# Ciphersuites

Each Transparency Log uses a single fixed ciphersuite, chosen when the log is
initially created, that specifies the following primitives to be used for
cryptographic computations:

* A hash algorithm
* A signature algorithm
* A Verifiable Random Function (VRF) algorithm

The hash algorithm is used for computing the intermediate and root values of
hash trees. The signature algorithm is used for signatures from both the service
operator and the third-party, if one is present. The VRF is used for preserving
the privacy of lookup keys. One of the VRF algorithms from {{!I-D.irtf-cfrg-vrf}} must be used.

Ciphersuites are represented with the CipherSuite type. The ciphersuites are
defined in {{kt-ciphersuites}}.


# Cryptographic Computations

## Commitment

Commitments are computed with HMAC {{RFC2104}}, using the hash function
specified by the ciphersuite. To produce a new commitment to a value called
`message`, the application generates a random 16 byte value called `opening` and
then computes:

~~~ pseudocode
commitment = HMAC(fixedKey, opening || message)
~~~

where `fixedKey` is the 16 byte hex-decoded value:

~~~
d821f8790d97709796b4d7903357c3f5
~~~

The output value `commitment` may be published, while `opening` and `message`
should be kept private until the commitment is meant to be revealed.

## Prefix Tree

The leaf nodes of a prefix tree are serialized as:

~~~ tls
struct {
    opaque key<VRF.Nh>;
    uint32 counter;
} PrefixLeaf;
~~~

where `key` is the full search key, `counter` is the counter of times that the
key has been updated (starting at 0 for a key that was just created), and
`VRF.Nh` is the output size of the ciphersuite VRF in bytes.

The parent nodes of a prefix tree are serialized as:

~~~ tls
struct {
  opaque value<Hash.Nh>;
} PrefixParent;
~~~

where `Hash.Nh` is the output length of the ciphersuite hash function. The value
of a parent node is computed by hashing together the values of its left and
right children:

~~~ pseudocode
parent.value = Hash(0x01 ||
                   nodeValue(parent.leftChild) ||
                   nodeValue(parent.rightChild))

nodeValue(node):
  if node.type == emptyNode:
    return make([]byte, Hash.Nh)
  else if node.type == leafNode:
    return Hash(0x00 || node.key || node.counter)
  else if node.type == parentNode:
    return node.value
~~~

where `Hash` denotes the ciphersuite hash function.

## Log Tree {#crypto-log-tree}

The leaf and parent nodes of a log tree are serialized as:

~~~ tls
struct {
  opaque commitment<Hash.Nh>;
  opaque prefix_tree<Hash.Nh>;
} LogLeaf;

struct {
  opaque value<Hash.Nh>;
} LogParent;
~~~

The value of a parent node is computed by hashing together the values of its
left and right children:

~~~ pseudocode
parent.value = Hash(hashContent(parent.leftChild) ||
                    hashContent(parent.rightChild))

hashContent(node):
  if node.type == leafNode:
    return 0x00 || nodeValue(node)
  else if node.type == parentNode:
    return 0x01 || nodeValue(node)

nodeValue(node):
  if node.type == leafNode:
    return Hash(node.commitment || node.prefix_tree)
  else if node.type == parentNode:
    return parent.value
~~~

## Tree Head Signature

The head of a Transparency Log, which represents the log's most recent state, is
represented as:

~~~ tls
struct {
  uint64 tree_size;
  uint64 timestamp;
  opaque signature<0..2^16-1>;
} TreeHead;
~~~

where `tree_size` counts the number of entries in the log tree and `timestamp`
is the time that the structure was generated, in milliseconds since the Unix
epoch. If the Transparency Log is deployed with Third-party Management then the
public key used to verify the signature belongs to the third-party manager;
otherwise the public key used belongs to the service operator.

The signature itself is computed over a `TreeHeadTBS` structure, which
incorporates the log's current state as well as long-term log configuration:

~~~ tls
enum {
  reserved(0),
  contactMonitoring(1),
  thirdPartyManagement(2),
  thirdPartyAuditing(3),
} DeploymentMode;

struct {
  CipherSuite ciphersuite;
  DeploymentMode mode;
  opaque signature_public_key<0..2^16-1>;
  opaque vrf_public_key<0..2^16-1>;

  select (Configuration.mode) {
    case contactMonitoring:
    case thirdPartyManagement:
      opaque leaf_public_key<0..2^16-1>;
    case thirdPartyAuditing:
      opaque auditor_public_key<0..2^16-1>;
  };
} Configuration;

struct {
  Configuration config;
  uint64 tree_size;
  uint64 timestamp;
  opaque root_value<Hash.Nh>;
} TreeHeadTBS;
~~~


# Tree Proofs

## Log Tree

An inclusion proof for a single leaf in a log tree is given by providing the
copath values of a leaf. Similarly, a bulk inclusion proof for any number of
leaves is given by providing the fewest node values that can be hashed together
with the specified leaves to produce the root value. Such a proof is encoded as:

~~~ tls
opaque NodeValue<Hash.Nh>;

struct {
  NodeValue elements<0..2^16-1>;
} InclusionProof;
~~~

Each `NodeValue` is a uniform size, computed by passing the relevent `LogLeaf`
or `LogParent` structures through the `nodeValue` function in
{{crypto-log-tree}}. Finally, the contents of the `elements` array is kept in
left-to-right order: if a node is present in the root's left subtree, it's value
must be listed before any values provided from nodes that are in the root's
right subtree, and so on recursively.

Consistency proofs are encoded similarly:

~~~ tls
struct {
  NodeValue elements<0..2^16-1>;
} ConsistencyProof;
~~~

Again, each `NodeValue` is computed by passing the relevent `LogLeaf` or
`LogParent` structure through the `nodeValue` function. The nodes chosen
correspond to those output by the algorithm in section 2.1.2 of {{RFC6962}}.

## Prefix Tree

A proof from a prefix tree authenticates that a search was done correctly for a
given search key. Such a proof is encoded as:

~~~ tls
enum {
  reserved(0),
  inclusion(1),
  nonInclusionLeaf(2),
  nonInclusionParent(3),
} PrefixSearchResult;

struct {
  PrefixSearchResult result;
  NodeValue elements<0..2^16-1>;
  select (PrefixProof.result) {
    case inclusion:
      uint32 counter;
    case nonInclusionLeaf:
      PrefixLeaf leaf;
    case nonInclusionParent:
  };
} PrefixProof;
~~~

The `result` field indicates what the terminal node of the search was:

- `inclusion` for a leaf node matching the requested key
- `nonInclusionLeaf` for a leaf node not matching the requested key
- `nonInclusionParent` for a parent node that lacks the desired child

The `elements` array consists of the copath of the terminal node, in
bottom-to-top order. That is, the terminal node's sibling would be first,
followed by the terminal node's parent's sibling, and so on. In the event that a
node is not present, an all-zero byte string of length `Hash.Nh` is listed
instead.

Depending on the `result` field, any additional information about the terminal
node that's necessary to verify the proof is also provided. In the case of
`nonInclusionParent`, no additional information is needed because the
non-terminal child's value is already in `elements`.

The proof is verified by hashing together the provided elements, in the
left/right arrangement dictated by the search key, and checking that the result
equals the root value of the prefix tree.

## Combined Tree {#proof-combined-tree}

A proof from a combined log and prefix tree follows the execution of a binary
search through the leaves of the log tree, as described in {{combined-tree}}. It
is serialized as follows:

~~~ tls
struct {
  PrefixProof prefix_proof;
  opaque commitment<Hash.Nh>;
} SearchStep;

struct {
  SearchStep steps<0..2^8-1>;
  InclusionProof inclusion;
} SearchProof;
~~~

Each `SearchStep` structure in `steps` is one leaf that was inspected as part of
the binary search, starting with the "middle" leaf (i.e., the leaf whose index
is the largest power of two less than the size of the log). The `prefix_proof`
field of a `SearchStep` is the output of searching the prefix tree whose root is
at that leaf for the search key, while the `commitment` field is the commitment
to the update at that leaf. The `inclusion` field of `SearchProof` contains a
batch inclusion proof for all of the leaves accessed by the binary search,
relating them to the root of the log tree.

A verifier interprets the output of each `prefix_proof` as `-1` if it is a
non-inclusion proof, or as `counter` if it is an inclusion proof. The proof can
then be verified by checking that:

1. The elements of `steps` represent a monotonic series over the leaves of the
   log, and
2. The elements of `steps` correspond exactly to the lookups that would be made
   by the verifier while conducting the binary search.

Once the validity of the search steps has been established, the verifier can
compute the root of each prefix tree represented by a `prefix_proof` and combine
it with the corresponding `commitment` to obtain the value of each leaf. These
leaf values can then be combined with the proof in `inclusion` to check that the
output matches the root of the log tree.


# Update Format

The updates committed to by a combined tree structure contain the new value of a
search key, along with additional information depending on the deployment mode
of the Transparency Log. They are serialized as follows:

~~~ tls
struct {
  select (Configuration.mode) {
    case contactMonitoring:
    case thirdPartyManagement:
      opaque signature<0..2^16-1>;
    case thirdPartyAuditing:
  };
} UpdatePrefix;

struct {
  UpdatePrefix prefix;
  opaque value<0..2^32-1>;
} UpdateValue;
~~~

The `value` field contains the new value of the search key.

In the event that third-party management is used, the `prefix` field contains a
signature from the service operator, using the public key from
`Configuration.leaf_public_key`, over the following structure:

~~~ tls
struct {
  opaque search_key<0..2^16-1>;
  uint32 version;
  opaque value<0..2^32-1>;
} UpdateTBS;
~~~

The `search_key` field contains the search key being updated, `version` contains
the new key version, and `value` contains the same contents as
`UpdateValue.value`. Clients MUST successfully verify this signature before
consuming `UpdateValue.value`.


# User Operations

## Search

Users initiate a Search operation by submitting a SearchRequest to the
Transparency Log containing the key that they're interested in. Users can
optionally specify a version of the key that they'd like to receive, if not the
most recent one. They can also include the `tree_size` of the last TreeHead that
they successfully verified.

~~~ tls
struct {
  opaque search_key<0..2^16-1>;
  optional<uint32> version;
  optional<uint64> last;
} SearchRequest;
~~~

In turn, the Transparency Log responds with a SearchResult structure:

~~~ tls
struct {
  TreeHead tree_head;
  optional<ConsistencyProof> consistency;
  select (Configuration.mode) {
    case contactMonitoring:
    case thirdPartyManagement:
    case thirdPartyAuditing:
      AuditorTreeHead auditor_tree_head;
  };
} FullTreeHead;

struct {
  opaque index<VRF.Nh>;
  opaque proof<0..2^16-1>;
} VRFResult;

struct {
  opaque opening<16>;
  opaque value<0..2^32-1>;
} SearchValue;

struct {
  FullTreeHead full_tree_head;
  VRFResult vrf_result;
  SearchProof search;

  optional<SearchValue> value;
} SearchResult;
~~~

If `last` is present, then the Transparency Log MUST provide a consistency proof
between the current tree and the tree when it was this size, in the
`consistency` field of `FullTreeHead`.

Users verify a search result by following these steps:

1. Verify the VRF proof in `VRFResult.proof` against the requested search key
   `SearchRequest.search_key` and the claimed VRF output `VRFResult.index`.
2. Evaluate the search proof in `search` according to the steps in
   {{proof-combined-tree}}. This will produce a verdict as to whether the search
   was executed correctly, and also a candidate root value for the tree. If it's
   determined that the search was executed incorrectly, abort with an error.
3. With the candidate root value for the tree:
   1. Verify the proof in `FullTreeHead.consistency`, if one is expected.
   2. Verify the signature in `TreeHead.signature`.
   3. Verify that the timestamp in `TreeHead` is sufficiently recent.
      Additionally, verify that the `timestamp` and `tree_size` fields of the
      `TreeHead` are greater than or equal to what they were before.
   4. If third-party auditing is used, verify `auditor_tree_head` with the steps
      described in {{auditing}}.
4. If the proof in `search` determined that a valid entry was found, check that
   `value` is populated, and that the commitment in the terminal search step
   opens to `SearchValue.value` with opening `SearchValue.opening`. If the proof
   determined that a valid entry was not found, check that `value` is empty.

Provided that the above verification is successful, users decode the encoded
`UpdateValue` structure in `SearchValue.value`. Depending on the deployment mode
of the Transparency Log, the `UpdateValue` may or may not require additional
verification, specified in {{update-format}}, before its contents may be
consumed.

## Update

Users initiate an Update operation by submitting an UpdateRequest to the
Transparency Log containing the new key and value to store. Users can also
optionally include the `tree_size` of the last TreeHead that they successfully
verified.

~~~ tls
struct {
  opaque search_key<0..2^16-1>;
  opaque value<0..2^32-1>;
  optional<uint64> last;
} UpdateRequest;
~~~

If the request is valid, the Transparency Log adds the new key-value pair to the
log and returns an UpdateResult structure:

~~~ tls
struct {
  FullTreeHead full_tree_head;
  VRFResult vrf_result;
  SearchProof search;

  opaque opening<16>;
  UpdatePrefix prefix;
} UpdateResult;
~~~

Users verify the UpdateResult as if it were a SearchResult for the most recent
version of `search_key`.

Note that the contents of `UpdateRequest.value` is the new value of the lookup
key and NOT a serialized `UpdateValue` object. To aid verification, the update
result provides the `UpdatePrefix` structure necessary to reconstruct the
`UpdateValue`.

## Monitor

Applications MUST retain the most recent `TreeHead` they've successfully
verified as part of a query result, and populate the `last` field of any query
request with the `tree_size` from this `TreeHead`.


# Third Parties

## Management

With the Third-party Management deployment mode, a third-party is responsible
for the majority of the work of storing and operating the log, while the service
operator serves mainly to enforce access control and authenticate the addition
of new entries to the log. All user queries specified in {{user-operations}} are
initially sent by users directly to the service operator, and the service
operator proxies them to the third-party manager if they pass access control.

The service operator only maintains one private key that is kept secret from the
third-party manager, which is the private key corresponding to
`Configuration.leaf_public_key`. This private key is used to sign new entries
before they're added to the log.

As such, all requests and their corresponding responses from {{user-operations}}
are proxied between the user and the third-party manager unchanged with the
exception of `UpdateRequest`, which needs to carry the service operator's
signature over the update:

~~~ tls
struct {
  UpdateRequest request;
  opaque signature<0..2^16-1>;
} ManagerUpdateRequest;
~~~

The signature is computed over the `UpdateTBS` structure from {{update-format}}.

## Auditing

With the Third-party Auditing deployment mode, the service operator obtains
signatures from a lightweight third-party auditor attesting to the fact that the
service operator is constructing the tree correctly. These signatures are
provided to users along with the results of their queries.

The third-party auditor is expected to run asynchronously, downloading and
authenticating a log's contents in the background, so as not to become a
bottleneck for the service operator. This means that the signatures from the
auditor will usually be somewhat delayed. Applications MUST specify a
maximum amount of time after which an auditor signature will no longer be
accepted. It MUST also specify a maximum number of entries that an auditor's
signature may be behind the most recent `TreeHead` before it will no longer be
accepted. Failing to verify an auditor's signature in a query MUST result in an
error that prevent's the query's result from being consumed or accepted by the
application.

The service operator submits updates to the auditor in batches, in the order
that they were added to the log tree:

~~~ tls
struct {
  opaque index<VRF.Nh>;
  opaque commitment<Hash.Nh>;
} AuditorUpdate;

struct {
  AuditorUpdate updates<0..2^16-1>;
} AuditorRequest;
~~~

The `index` field of each `AuditorUpdate` contains the VRF output of the search
key that was updated and `commitment` contains the service provider's
commitment to the update. The auditor responds with:

~~~
struct {
  TreeHead tree_head;
} AuditorResponse;
~~~

The `tree_head` field contains a signature from the auditor's private key,
corresponding to `Configuration.auditor_public_key`, over the serialized
`TreeHeadTBS` structure. The `tree_size` field of the `TreeHead` is equal to the
number of entries processed by the auditor and the `timestamp` field is set to
the time the signature was produced (in milliseconds since the Unix epoch).

The auditor `TreeHead` from this response is provided to users wrapped in the
following struct:

~~~ tls
struct {
  TreeHead tree_head;
  opaque root_value<Hash.Nh>;
  ConsistencyProof consistency;
} AuditorTreeHead;
~~~

The `root_value` field contains the root hash of the tree at the point that the
signature was produced and `consistency` contains a consistency proof between
the tree at this point and the most recent `TreeHead` provided by the service
operator.

To check that an `AuditorTreeHead` structure is valid, users follow these steps:

1. Verify the signature in `TreeHead.signature`.
2. Verify that `TreeHead.timestamp` is sufficiently recent.
3. Verify that `TreeHead.tree_size` is sufficiently close to the most recent
   tree head from the service operator.
4. Verify the consistency proof `consistency` between this tree head and the
   most recent tree head from the service operator.


# Owner Signing

TODO


# Out-of-Band Communication

TODO


# Security Considerations

While providing a formal security proof is outside the scope of this document,
this section attempts to explain the intuition behind the security of each
deployment mode.

## Contact Monitoring {#security-contact-monitoring}

Contact Monitoring works by splitting the monitoring burden between both the
owner of a key and those that look it up. Stated as simply as possible, the
monitoring obligations of each party are:

1. The key owner, on a regular basis, searches for the most recent version of
   the key in the log. They verify that this search results in the expected
   version of the key, at the expected position in the log.
2. The user that looks up a key, whenever a new parent node is established on
   the key's direct path, searches for the key in the prefix tree stored in this
   new parent node. They verify that the version counter returned is greater
   than or equal to the expected version.

To understand why this is secure, we look at what happens when the service
operator tampers with the log in different ways.

First, say that the service operator attempts to cover up the latest version of
a key, with then goal of causing a "most recent version" search for the key to
resolve in a lower version. To do this, the service operator must add a parent
node over the latest version of the key with a prefix tree that contains an
incorrect version counter. Left unchanged, the key owner will observe that the
most recent version of their key is no longer available the next time they
perform monitoring. Alternatively, the service operator could add the new
version of the key back at a later position in the log. But even so, the key
owner will observe that the key's position has changed the next time they
perform monitoring. The service operator is unable to restore the latest version
of the key without violating the log's append-only property or presenting a
forked view of the log to different users.

Second, say that the service operator attempts to present a fake new version of
a key, with the goal of causing a "most recent version" search for the key to
resolve to the fake version. To do this, the service operator can simply add the
new version of the key as the most recent entry to the log, with the next
highest version counter. Left unchanged, or if the log continues to be
constructed correctly, the key owner will observe that a new version of their
key has been added without their permission the next time they perform
monitoring. Alternatively, the service operator can add a parent node over the
fake version with an incorrect version counter to attempt to conceal the
existence of the fake entry. However, the user that previously consumed the fake
version of the key will detect this attempt at concealment the next time they
perform monitoring.

## Third-party Management

Third-party Management works by separating the construction of the log from the
ability to approve which new entries are added to the log, such that tricking users
into accepting malicious data requires the collusion of both parties.

The service operator maintains a private key that signs new entries before
they're added to the log, which means that it has the ability to sign malicious
new entries and have them successfully published. However, without the collusion
of the third-party manager to later conceal those entries by constructing the
tree incorrectly, their existence will be apparent to the key owner the next
time they perform monitoring.

Similarly, while the third-party manager has the ability to construct the tree
incorrectly, it cannot add new entries on its own without the collusion of the
service operator. Without access to the service operator's signing key, the
third-party manager can only attempt to selectively conceal the latest version
of a key from certain users. However, as discussed in
{{security-contact-monitoring}}, this is also apparent to the key owner through
monitoring.


## Third-party Auditing

Third-party Auditing works by requiring users to verify a signature from a
third-party auditor attesting to the fact that the service operator has been
constructing the tree correctly.

While the service operator can still construct the tree incorrectly and
temporarily trick users into accepting malicious data, an honest auditor will no
longer provide its signatures over the tree at this point. Once there are no
longer any sufficiently recent auditor tree roots, the log will become
non-functional as the service operator won't be able to produce any query
results that would be accepted by users.


# IANA Considerations

This document requests the creation of the following new IANA registries:

* KT Ciphersuites ({{kt-ciphersuites}})

All of these registries should be under a heading of "Key Transparency",
and assignments are made via the Specification Required policy {{!RFC8126}}. See
{{de}} for additional information about the KT Designated Experts (DEs).

RFC EDITOR: Please replace XXXX throughout with the RFC number assigned to
this document

## KT Ciphersuites

TODO

## KT Designated Expert Pool {#de}

TODO


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
