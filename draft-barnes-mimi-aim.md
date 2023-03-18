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

# ActivityPub

    * ActivityPub has C2S and S2S protocols — S2S meaningful here
    * ActivityPub ~= SMTP + HTTP
        * Messages are represented as Activities
        * “Activities” have are addressed to “Actors” (“to” / “cc” / “target” / etc.)
        * “Actors” have “Inboxes”
        * Originating server POSTs Activity to Inboxes of recipient Actors
    * Activities also have other useful attributes, e.g., inReplyTo, tags
    * Identity
        * Actor = URL
        * Some ActivityPub applications use WebFinger to resolve acct:user@domain
        * “acct” URIs are implicit, render as @user@domain or just @user

# Using ActivityPub for MIMI

## Identifying Users

    * `acct` + WebFinger

## Delivery of Direct Messages

    * Realizing DMs is natural, just use the email-like addressing

## Delivery of Messages in a Channel

    * Channel = destination
    * Channel owner server does fanout
    * alsoKnownAs enables transferability

## End-to-End Security

    * Actors have profiles, which can be extended to expose KeyPackages
    * If users are identified with acct URIs, the identified user is a natural identity authority
    * No centralized authority for sequencing => client-side resolution algorithm + a degree of YOLO
    * Transmit MLS messages as activities
        * => identify MLS group+epoch by an activity URL
        * => identify predecessor epochs by inReplyTo
        * => have Activity link to the MLS group+epoch with which it was protected

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

This investigation was inspired by a [Mastodon post by Darius
Kazemi](https://friend.camp/@darius/109996157569528129).


