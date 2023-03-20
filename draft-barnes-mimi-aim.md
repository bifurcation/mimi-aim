---
title: "ActivityPub for Interoperable Messaging (AIM)"
abbrev: "AIM"
category: info

docname: draft-barnes-mimi-aim-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "More Instant Messaging Interoperability"
keyword:
 - MIMI
 - messaging
 - ActivityPub
 - end-to-end encryption
venue:
  group: "More Instant Messaging Interoperability"
  type: "Working Group"
  mail: "mimi@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/mimi/"
  github: "bifurcation/mimi-aim"
  latest: "https://bifurcation.github.io/mimi-aim/draft-barnes-mimi-aim.html"

author:
 -
    fullname: Richard L. Barnes
    organization: Cisco
    email: "rlb@ipv.sx"

normative:
  W3C.ActivityPub:
  RFC7565:
  RFC7033:

informative:
  RFC561:
  RFC6120:
  Mastodon:
    target: https://docs.joinmastodon.org/spec/activitypub/
    title: "Mastodon"

--- abstract

The MIMI working group is chartered to define tools that messaging providers can
use to interoperate with one another.  The W3C ActivityPub protocol is already
widely used for several use cases that resemble the MIMI use case.  This
document examines whether ActivityPub might be a good baseline for providing the
sort of interoperability that MIMI intends to achieve.


--- middle

# Introduction

The MIMI working group is chartered to define tools that messaging providers can
use to interoperate with one another.  Messaging is obviously not a new
application; readers of ["Message Transmission Protocol"](#RFC561) from 1975 will find
familiar concepts such as "TO", "CC", and "BCC" fields.  It thus seems likely
that some existing protocol will satisfy many of MIMI's requirements.  Basing
MIMI on an existing widely deployed protocol can also facilitate deployment of
the MIMI protocol, since the lessons from deployment of the predecessor protocol
should mostly carry forward.

This document considers the W3C [ActivityPub protocol](#W3C.ActivityPub) as such
candidate to be such a baseline for MIMI.  At a high, level, ActivityPub is a
protocol for sharing "Activities" with various semantics among users homed to
loosely-coupled servers.  ActivityPub was published as a W3C Recommendation in
2018, and today supports several wide-scale services.  The largest and most
prominent of these is the [Mastodon microblogging platform](#Mastodon), which as
of this writing has around 10 million registered users, and an active userbase
in the millions.

The fact that Mastodon is based on ActivityPub is suggestive of how ActivityPub
might be useful for MIMI.  On the one hand, while Mastodon is primarily used for
distributing public messages, it also allows users to post private messages that
are only delivered to specific recipients.  On the other hand, Mastodon's focus
on wide distribution of public messages suggests that ActivityPub could support
messaging among large numbers of recipients.  Mastodon also includes some
extensions to ActiivityPub that could also be salient to MIMI, such as the use
of [`acct` URIs](#RFC7565) to identify users and the use of [WebFinger](#RFC7033) to resolve these URIs
to routable identifiers.

In the remainder of this document, we review the MIMI requirements and the
high-level architecture of ActivityPub.  We then sketch out approaches based on
ActivityPub for realizing the use cases salient to MIMI, highlighting both areas
where there is natural overlap between MIMI and ActivityPub and areas where
ActivityPub might need modification or extension to support MIMI use cases.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# MIMI Requirements

Summary:
* S2S federation, with the internals of each messaging system opaque
* Components of the system <— ActivityPub covers transport, identity
* Use cases:
    * 1:1 messaging
    * “Group DMs” ~ immutable, singular group <— works pretty naturally
    * Channels ~ variable membership, more reified <— not a seamless fit
* Ownership models?
    * Single-owner, permanent
    * Single-owner, transferable <— target this model
    * Multi-owner, synchronized
    * Mutli-owner, YOLO. <— or this model

## Components

An overall solution for interoperability between messaging services naturally
breaks down into a few components, as illustrated in {{fig-components}}:

* A **Transport** system that delivers messages between services, including
  enough information for the services to route the messages to the correct set
  of end clients.

* An **End-to-End (E2E) Security** layer that protects message contents from
  inspection or tampering by the services involved in delivering them.

* An **Identity** system that provides:
  * A client addressing scheme by which the servers participating in the
    transport can identify which clients should receive a message.
  * A credential scheme that is used to authenticate clients to one another in
    the end-to-end security system.

* Formats for messages carried within end-to-end protection that enable
  **Messaging** and **Real-Time** applications.

~~~ aasvg
                    +-----------+-----------+
                    | Messaging | Real-Time |
+----------+        +-----------+-----------+
| Identity +---+--->+     E2E Security      |
+----------+   |    +-----------------------+
               +--->+       Transport       |
                    +-----------------------+
~~~
{: #fig-components title="Components of MIMI" }

In other words, the E2E security layer creates a demarcation between things that
are visible to servers and things that are not.  The transport protocol defines
the former, message formats the latter.

## Service-to-Service Interoperability

MIMI is focused on interoperability between messaging **services**.  Unlike
earlier messaging protocols like [XMPP](#RFC6120), which cover client-to-server
interactions as well as server-to-server interactions, MIMI is focused primarily
on the latter.

~~~ aasvg
        Domain A      MIMI Transport        Domain B
           |                 |                 |
    .-----' '-----.   .-----' '-----.   .-----' '-----.
   |               | |               | |               |
Client A <-----> Service A <-----> Service B <-----> Client B

~~~
{: #s2s title="MIMI delivers messages between services" }

The MIMI transport system and the routing functions of the identity system
operate within the inter-service interaction.  The services are presumed to be
able to deliver messages to connected clients based on information provided by
the transport system.

As the name implies, the E2E security system must be compatible across the
various clients that comprise the endpoints of a messaging interaction.  This in
turn requires that the authentication aspects of the identity layer and the
message formats be understood by clients.  Since these components are not
accessible to servers (due to E2E protections), they need to be handled locally
on clients.

Here again, the E2E security layer creates a demarcation, between protocol
features that are server-to-server and client-to-client scoped.  Note, however,
that no part of the protocol covers client-to-server interactions.  These are
the domain of the individual services.

## Transport Use Cases

The messaging applications among which MIMI is to provide interoperability
typically support two types of interaction with complementary properties:

* Group Direct Messages (DMs): The interaction has a static set of participants,
  and is "singular", in the sense that any direct message to exactly that set of
  participants is presumed to belong to the interaction.

* Channels: An interaction with a dynamic set of participants.  Multiple
  channels can have the same set of participants, and participants can join and
  leave the channel.

(These concepts have various names in different messaging systems.  The naming
here is not intended to indicate alignment with one system over another, but to
choose some common terminology with appropriate connotations.)

Many systems also support one-to-one messaging, but this can be considered a
special cases of Group DMs, in the sense that one-to-one conversations are
typically singular interactions with have a static participant set.  It is also
common for an interaction that appears to be 1:1 in a user interface to be
realized with group messaging, for example to accommodate users' use of multiple
devices.

One way to view the distinction between group DMs and channels is that in a
channel-style interaction, the interaction is "reified", in the sense that it is
an entity in the protocol that can be the subject of metadata, the object of
actions, etc.  Group DMs, on the other hand, are defined only by their
participant list.  Channels are like XMPP MUCs {{?RFC7702}}; group DMs are more
like email.

# ActivityPub

In this section, we provide a brief overview of the ActivityPub protocol.
ActivityPub defines both client-to-server and server-to-server protocols.
Because the MIMI transport layer only goes between two services, we focus on the
server-to-server protocol.

At a very high level, ActivityPub is similar to SMTP with JSON messages and
delivery over HTTP.  An ActivityPub server forwards messages from local clients to
their intended recipients, and receives messages from other servers intended for
its local clients.  An ActivityPub server also makes available metadata that
support the functioning of the protocol.

## Actors and Activities

The main entities in ActivityPub are Actors and Activities.  In most cases, an
Actor represents a user (`"type": "Person"`), but Actors can also represent
automated services (`"type": "Service"`) or collections of other Actors
(`"type": "Group"`).  Each Actor has a unique URI, from which a JSON-LD
description of the Actor's attributes can be retrieved.  {{fig-actor}} shows a
simple Actor description.

Activities represent a variety of actions within the system, including "Create"
activities that carry new messages as well as things like "Add" and "Remove" to
allow modifications of collections.  {{fig-activity}} shows a Create activity
that reflects the creation of a new Note object.  Activities can also carry
metadata such as `inReplyTo` or `tags`.

~~~ json
{
  "@context": ["https://www.w3.org/ns/activitystreams"],
  "type": "Person",
  "id": "https://example.com/users/alice",
  "inbox": "https://example.com/users/alice/inbox",
  "outbox": "https://example.com/users/alice/feed"
}
~~~
{: #fig-actor title="A sample Actor" }

~~~ json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Create",
  "id": "https://example.net/~mallory/87374",
  "actor": "https://example.net/~mallory",
  "object": {
    "id": "https://example.com/~mallory/note/72",
    "type": "Note",
    "attributedTo": "https://example.net/~mallory",
    "content": "This is a note",
    "published": "2015-02-10T15:04:55Z",
    "to": ["https://example.org/~john/"],
    "cc": ["https://example.com/~erik/followers"]
  },
  "published": "2015-02-10T15:04:55Z",
  "to": ["https://example.org/~john/"],
  "cc": ["https://example.com/~erik/followers"]
}
~~~
{: #fig-activity title="A sample Activity" }

## Activity Delivery

Delivery of activities in ActivityPub follows a push pattern, with the ability
to pull messages as a fallback.

Each Actor has a "inbox" and "outbox" URIs, which allow external parties to
deliver Activities to the Actor or read Activities that the Actor has posted,
respectively.

To send an Activity to the Actor, a remote server sends an HTTP POST request to
the Actor's inbox URI. When a client of an ActivityPub server asks it to
distribute an Activity, the server identifies the set of Actors that are the
intended recipients of the Activity (e.g., using the `to` and `cc` fields
visible in {{fig-activity}}), and sends POSTs requests containing the activity
to the Actors' inboxes.

Outbox URIs allow a remote server to query the list of Activities that the Actor
has posted. To read Activities posted by the actor, a remote party sednes an
HTTP GET request to the outbox URL.  The outbox includes a paging function to
allow traversal of large sets of Activities.

Both inbox and outbox requests are constrained by an authorization model, so
that a server can constrain which Actors allowed to communicate.

## Identity

The native identifiers for ActivityPub are Actor URIs. These URIs are HTTP URIs
that both identify end users as well as services and groups and allow the
metadata for the Actor to be retrieved.

HTTP URIs are of course not a very user-friendly identifier.  So many
applications based on ActivityPub use identifiers of the form `@username@domain`
or simply `@username` when the domain is clear from context. These identifiers
represent `acct` URIs {{RFC7565}}, which, in the words of the RFC, "identify a
user's account at a service provider, irrespective of the particular protocols
that can be used to interact with the account".

In order to engage in ActivityPub interactions with an Actor given such an
identifier, the application resolves the identifier to an Actor URI using
WebFinger {{RFC7033}}. For example, given the URI `acct:alice@example.com`, the
application would send a GET request to
`https://example.com/.well-known/webfinger?resource=acct:alice@example.com`.
The response would indicate various contact points associated with that account,
as shown in {{fig-webfinger}}.  The ActivityPub Actor URI is indicated by the
`href` in the `links` entry with `"type": "application/activity+json"`.

~~~ json
{
  "subject": "acct:alice@example.com",
  "links": [
    {
      "rel": "http://webfinger.net/rel/profile-page",
      "type": "text/html",
      "href": "https://example.com/@alice"
    },
    {
      "rel": "self",
      "type": "application/activity+json",
      "href": "https://example.com/users/alice"
    }
  ]
}
~~~
{: #fig-webfinger title="A WebFinger response for `acct:alice@example.com`" }

# Using ActivityPub for MIMI

In this document, we consider the use of ActivityPub and related technologies
for the transport and identity systems, and the integration of MLS for the E2E
security layer {{!I-D.ietf-mls-protocol}}.  Message formats are not handled
here.

Points at which ActivityPub would need to be extended are highlighted with
**[EXT]**.  These are the domains where the MIMI working group would need to
define protocol extensions to build an overall messaging systme based on
ActivityPub.

## User Identity and Metadata

The primary identifier for a user is an `acct` URI, which is resolved to an
Actor URI using WebFinger as described in {{identity}}.

Aside from UI considerations, this choice of primary identifier is important for
authentication at the end-to-end security layer.  An `acct` URI is a scoped
identifier, in the sense that the domain is the authoritative source of
information about what entity is represented by of the user portion of the URI.
Indeed, this is the whole premise of using WebFinger for `acct` URI resolution.

**[EXT]** To leverage this information in an MLS-based end-to-end security
layer, all that is needed is a credential issued by the domain that attests that
the holder of a given signature key legitimately represents the user portion of
hte URI, for example an X.509 certificate or Verifiable Credential {{?RFC5280}}
{{?W3C.vc-data-model}}.  MIMI would need to verify the format for such
credentials and how a client receiving one would verify it, but would not need
to specify an issuance API.  However, given that domains are already assumed to
know how to authenticate their users, such an API could be as simple as a single
authenticated POST request containing a proof of control of a key pair, whose
response would then contain the desired credential.

The ActivityPub Actor object contains optional fields that can provide
additional metadata about a user, for example a profile URL or preferred
username.

**[EXT]** The Actor object would be a convenient mechanism to distribute the
cryptographic material required to initiate end-to-end secure communications
with an actor, i.e., KeyPackage objects in the case of MLS.  This facility would
be slightly more complicated than the static metadata fields currently present.
KeyPackages are intended to be single-use, so the server managing the Actor
object would need to selectively provide different KeyPackages in response to
differnet queries.  Multi-device scenarios might require multiple KeyPackages to
be provided in response to a single query.

## Channels

The channel use case can be implemented by representing the channel as an
ActivityPub Actor.  Metadata related to the channel can be published and managed
as part of the Actor object.  In particular, the `followers` collection for the
Actor can be used to track the membership of the channel, so that normal
ActivityPub patterns can be followed for message delivery and membership
management.

**[EXT]** A channel's Actor also tracks information about the end-to-end
security state of the channel.  For MLS, this would entail tracking information
about an MLS group associated to the channel, most importantly the current epoch
and ratchet tree.  A channel may also need to store a GroupInfo object for the
group, as discussed in {{membership-and-metadata-management}}.

In the context of federated messaging, the question of which server hosts a
channel could be contentious.  For example, if Alice creates a channel on
Service A and invites Bob and Charlie from Services B and C, but then Alice
leaves the channel, does Service A continue to host the channel even though none
of their users are involved?  If this is a problem the working group needs to
tackle, it will likely be useful to follow the approaches used in Mastodon for
moving or linking accounts, e.g., using a `Move` activity.

### Channel Creation

When a channel is created on a service's server by a user of that server, no
MIMI/ActivityPub action is needed.  The server hosting the channel can notify
the members of the channel that it has been created by sending a `Create`
activity to their inboxes.

~~~ json
{
  "@context": [
    "https://www.w3.org/ns/activitystreams",
    "urn:ietf:ns:mimi",
  ],
  "summary": "Alice created a channel",
  "type": "Create",
  "id": "http://example.com/activities/1",
  "actor": "http://example.com/user/alice",
  "object": {
    "type": "Service",
    "id": "http://example.com/channels/e4f70622",
    "name": "MIMI discussion group"
  },
  "mimi:welcome": "<base64-encoded Welcome>",
  "to": "http://john.example.org"
}
~~~
{: #fig-create title="A Create activity announcing a new channel" }

**[EXT]** To set up the end-to-end security for the channel, the creator of the
channel will need to fetch KeyPackages for the other members of the channel.
For members using other services, KeyPackages can be fetched via the members'
Actor objects, as discussed in {{user-identity-and-metadata}}.  An MLS Welcome
message enabling the members to initialize their MLS state is attached to the
`Create` activity.

### Message Delivery

Messages are sent within a channel by sending a `Create` activity to the channel
Actor's inbox, addressed to the channel's followers.  Following the "Forwarding
from inbox" pattern discussed in {{W3C.ActivityPub}}, the server hosting the
channel will then forward the activity to inboxes of the members of the channel.
The message content itself is an MLS PrivateMessage encapsulating the actual
content to be delivered to the channel.

~~~ json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Create",
  "id": "https://example.net/~mallory/87374",
  "actor": "https://example.net/~mallory",
  "object": {
    "id": "https://example.com/~mallory/note/72",
    "type": "Note",
    "attributedTo": "https://example.net/~mallory",
    "content": "<base64 encoded MLS PrivateMessage>",
    "published": "2015-02-10T15:04:55Z",
    "to": ["http://example.com/channels/e4f70622/followers"]
  },
  "published": "2015-02-10T15:04:55Z",
  "to": ["http://example.com/channels/e4f70622/followers"]
}
~~~
{: fig-message title="A Create activity sending a message to a channel" }

If the members have a `sharedInbox` field in their Actor objects, this delivery
can be quite efficient at the inter-service level: Only one copy of the activity
will be sent to each shared inbox, effectively once per service involved in the
channel.

### Membership and Metadata Management

Members of the channel add and remove other members by using `Add` and `Remove`
activities to propose modifications to the followers collection associated to
the channel's Actor.  `Add` activities should be forward to the new member to
make them aware of their membership in the channel.

**[EXT]** An `Add` or `Remove` activity must include an MLS Commit that
implements the  corresponding action on the MLS group.  The Commit message must
be sent as a PublicMessage so that the server can update its representation of
the group's ratchet tree based on the content of the Commit.  An `Add` activity
must also include an MLS Welcome message allowing the new member to initialize
their MLS state.  Before accepting an `Add` or `Remove` activity for a channel,
the server must verify that the attached Commit corresponds to the current MLS
epoch for the channel, and reject the activity if this is not the case.

~~~ json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Add",
  "id": "https://example.com/user/alice/98485",
  "actor": "http://example.com/user/alice",
  "object": "http://example.net/user/bob",
  "target": "http://example.com/channels/e4f70622/followers",
  "commit": "<base64-encoded MLS Commit>",
  "welcome": "<base64-encoded MLS Welcome>",
  "to": ["http://example.com/channels/e4f70622/followers"]
}
~~~
{: fig-add title="An Add activity adding a new member to a channel" }

Other channel metadata (e.g., the name of the channel) can be updated by sending
an Update activity to the channel Actor's inbox.

~~~ json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Update",
  "actor": "http://example.com/user/alice",
  "object": "object": {
    "type": "Service",
    "id": "http://example.com/channels/e4f70622",
    "name": "MIMI discussion group (now with more ActivityPub!)"
  },
  "to": "http://example.com/channels/e4f70622/inbox"
}
~~~
{: #fig-update title"An Update activity changing the name of the channel" }

## Group DMs

In principle, since group DMs don't have any independent state aside from the
recipient list, groupDMs could be implemented directly using ActivityPub's
addressing model.  Activities could be directly addressed to other
actors using the `to` field, and a service receiving an activity could
associate it to a group DM based on the recipient list in the `to` field.

This approach would simplify certain things. For example, if a group DM used an
Actor for distribution as with a channel, it would be necessary to explicitly
enforce that there was only one such Actor per group DM; with direct
addresising, no such common resources are created, so there is no need to ensure
their uniqueness.  

Such a decentralized approach, however, does not work well with MLS, which works
best with a central coordination point to manage the sequencing of changes to
MLS state.  There are a couple of compromise options available here.

It might be feasible have the MLS groups attached to group DMs be immutable.
The first person to send a message in a group DM would include a Welcome
addressed to KeyPackages for all the other recipients.  That message would
initialize an MLS group including all the other recipients, which would be used
to protect further messages.

While the immutability approach is appealing in its simplicity, it might not be
workable.  Participants in the group DM might want to update their keys for
post-compromise security, or they might want to add a new device that they start
using after the group DM starts.  Both of these operations require changes to
the MLS group.

To allow mutable MLS groups, group DMs could use direct addressing for message
delivery, but link to an MLS group managed more like the MLS group attached to a
channel.

# Security Considerations

ActivityPub uses HTTPS for transport security on server-to-server interactions.

## End-to-End Security

{{using-activitypub-for-mimi}} includes provisions for implementing an end-to-end
security layer based on MLS.  As described in {{I-D.ietf-mls-protocol}}, MLS
requires a Delivery Service (DS) and an Authentication Service (AS) in order to
be integrated into an application.

Here, the DS functions are provided in a decentralized fashion by the
ActivityPub servers representing the interoperating services.  KeyPackages are
distributed via users' Actor objects (see
{{user-identity-and-metadata}}).  Other MLS messages are distributed as part of
membership management activities (see {{membership-and-metadata-management}}).

The AS function is provided by the service-issued credentials discussed in
{{user-identity-and-metadata}}.

## Forward Secrecy

By default, MLS provides forward secrecy and post-compromise security for
messages sent within a group.  In the most straightforward application of MLS to
messaging, this means that a new member of a channel will not be able to decrypt
messages from before they joined the group.  If providing access to historical
messages is a desired feature, than further mechanism will be required to
provide new members access to historical keys.

## Authentication and Authorization

There are some open questions here related to authentication and authorization,
for example:

* How should servers authenticate each other?

* How a receiving server knows that an Activity authentically comes from the
  Actor who is supposed to have sent it?

* What access control policies can a server enforce on inbound messages?

The ActivityPub specification is very light on details on these topics. However,
applications such as Mastodon have likely developed solutions that could be used
as starting points.

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

This investigation was inspired by a [Mastodon post by Darius
Kazemi](https://friend.camp/@darius/109996157569528129).


