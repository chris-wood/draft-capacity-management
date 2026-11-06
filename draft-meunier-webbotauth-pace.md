---
title: "Proportionate Access and Capacity Enforcement"
abbrev: "PACE"
category: info

docname: draft-meunier-webbotauth-pace-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: false
v: 3
# area: "Web and Internet Transport"
# workgroup: "Web Bot Auth"
keyword:
 - anonymous rate limiting
venue:
#  group: "Web Bot Auth"
#  type: "Working Group"
#  mail: "web-bot-auth@ietf.org"
#  arch: "https://mailarchive.ietf.org/arch/browse/web-bot-auth/"
  github: "thibmeu/draft-brian"
  latest: "https://thibmeu.github.io/draft-brian/draft-meunier-webbotauth-pace.html"

author:
 -
    fullname: Thibault Meunier
    organization: Cloudflare
    email: ot-ietf@thibault.uk
 -
    fullname: Christopher A. Wood
    organization: Cloudflare
    email: caw@heapingbits.net

normative:

informative:
  CIVIL: I-D.draft-meunier-bcp-civil
  AAUTH: I-D.draft-hardt-oauth-aauth-protocol
  CONNECT-UDP: RFC9298
  CONNECT-IP: RFC9484
  OHTTP: RFC9458
  PRIVACY-PARTITIONING: RFC9614
  HTTP: RFC9110
  HTTPSIG: RFC9421
  OAUTH: RFC6749
  BEARER: RFC6750
  OAUTH-RS-META: RFC9728
  STEPUP: RFC9470
  DPOP: RFC9449
  PRIVACYPASS: RFC9576
  PRIVATETOKEN: RFC9577
  OIDC:
    title: "OpenID Connect Core 1.0"
    target: "https://openid.net/specs/openid-connect-core-1_0.html"
  INTERNET-FOR-END-USER: RFC8890
  IPv6: RFC8200
  MOLE-ARCH: I-D.draft-jms-mole-architecture
  MOLE-PROTOCOLS: I-D.draft-jms-mole-protocols
  MOLE-HTTP: I-D.draft-jms-mole-http-transport
  OAUTH-XAA: I-D.draft-ietf-oauth-identity-assertion-authz-grant
  WAG: I-D.draft-carleton-workload-authz-grant
  POPPY:
    title: "Personal Agent Protocol, draft 0.3"
    target: "https://github.com/personal-agent-protocol/personal-agent-protocol/blob/1de1990e4978b42179fe0e3e0ae554271808c308/content/versions/0.3/spec.mdx"
  AAP:
    title: "Agent Attestation Protocol"
    target: "https://github.com/agentic-protocols/agentic-attestation-protocol/blob/1328a6bbfda24ea18bcd8996a5417a6702c8f274/draft-agent-attestation-protocol.md"
  USER-AGENT-HISTORY:
    title: "WebAIM: History of the browser user-agent string"
    target: "https://webaim.org/blog/user-agent-string-history/"
  WBA-ABA: I-D.draft-rescorla-anonymous-webbotauth
  WBA-HTTPSIG: I-D.draft-ietf-webbotauth-httpsig-protocol
  WIMSE-AIMS: I-D.draft-ietf-wimse-aims
  WIMSE-ARCH: I-D.draft-ietf-wimse-arch

...

--- abstract

Origins rate limit requests by address or by the platform they come through.
Behind a shared platform, such as an AI agent service, a VPN, or a network
operator, every user then shares the same fate. This document describes the
problem of limiting consumption without requiring users or their agents to
disclose a stable identifier. It sketches a direction in which Anchors
endorse agents, origins authorize those agents through anonymous budgets
rather than identities, and platforms identify themselves for separate
aggregate limits.

--- middle

# Introduction {#intro}

Agents are increasingly used in business and user workflows, including AI
assistants, search indexing, content aggregation, and automated testing. Their
relationship with origins is symbiotic. They need information from origins, and
origins need signals to apply rate-limit policies. These can be binary, like
allow or block when an origin cannot serve traffic, or resource-based to ensure
fair access to everyone.

Historically, most of these signals have been enshrined into deployments of
internet protocols. An agent needs an IP address to receive a response from an
origin. Its TLS stack and browser behavior provide further signals. It provides
a familiar User-Agent because middleboxes might otherwise filter its traffic
(see {{USER-AGENT-HISTORY}}).

Recent developments break these patterns. Agents use proxies
({{CONNECT-UDP}}, {{CONNECT-IP}}, {{OHTTP}}, or plain {{HTTP}}).
As platforms make requests on behalf of users, the user's TLS stack may no
longer be the one visible to the origin. AI agents, on the user's device or
hosted remotely, fetch and render information in place of browsers. User-Agent
strings are still spoofed.

Origins need to apply policy despite these changes. Many approaches rely on
obtaining a stable identifier. An address, including an {{IPv6}} address, is not
necessarily a user. Blocking a shared IP range can affect everyone behind it.
In enterprise settings, an agent can identify as a workload {{WIMSE-AIMS}}.
Platforms that can tolerate attribution can identify using {{WBA-HTTPSIG}}.
Other approaches involve user authentication or authorization, such as {{AAUTH}}
and {{OAUTH-XAA}}. {{system}} places these options side by side.

While identification solves many problems, it should not be necessary just to
limit an agent's consumption. The problem is to give origins that control
without requiring users or their agents to disclose a stable identifier. This
supports the user interests discussed in {{INTERNET-FOR-END-USER}}.

## Motivating use cases

In the cases below, traffic passes through platforms. Each user also has a
relationship with some Anchor, often but not necessarily that platform, which
can grant scarce Endorsements. Origins could use those to apply rate-limit
policies without learning who the user is.

1. AI agent services host remote agents and concentrate their traffic on a
   small set of IP addresses. One user's activity can affect others when an
   origin can only limit the shared address or the platform as a whole.
2. Network operators carry traffic from device-resident agents behind shared
   IP ranges. Policies applied to an address range or an entire ASN give these
   users a shared fate.
3. VPN providers and enterprise ZTNA gateways have a noisy neighbour problem.
   They may know their users, but should not have to expose them to each
   origin. An origin able to tell employees apart could single out an
   executive's activity for instance.

## Related work

{{WBA-ABA}} shares the anonymous rate-limiting goal and the MOLE construction.
This document is a companion to it. It focuses on platforms that already have
a relationship with their users, and on composing with a platform's
{{WBA-HTTPSIG}} identity.

Poppy {{POPPY}} uses User IDs and sessions, including for anonymous browsing.
{{AAP}} uses anonymous admission tokens and recommends a session afterwards,
though it also allows a fresh token per request. Here, the aim is to maintain a
per-agent budget without linking its presentations.

{{WIMSE-AIMS}} profiles WIMSE and OAuth for agents. It treats an agent as a
workload with its own identifier and credential, and carries the user, when
there is one, only as the subject of an OAuth access token, so that the two
identities are separate and compose on one request. {{WAG}} lets a
platform-hosted workload obtain such a token for itself without per-workload
registration. This document places both among the options an origin can ask
for, next to attribution and anonymous capacity.

# Conventions and Definitions {#terminology}

This document uses the HTTP terminology of {{HTTP}}, the MOLE terminology of
{{MOLE-ARCH}}, and the workload terminology of {{Section 2 of WIMSE-ARCH}}.
For instance:

User agent:
: A client program that initiates a request {{Section 3.5 of HTTP}}.

Intermediary:
: A proxy, gateway, or tunnel {{Section 3.7 of HTTP}}.

Origin server:
: A program that can originate authoritative responses for a target resource
  {{Section 3.6 of HTTP}}.

Workload:
: A logical software entity that may run as one or more workload instances
  {{Section 2 of WIMSE-ARCH}}.

Platform:
: A provider that hosts or carries traffic for agents.

Tenant:
: A customer that deploys workloads on a platform.

User:
: A person on whose behalf requests are made. A user has an existing
  relationship with an Anchor, such as an account, a subscription, or an
  employment contract.

Agent:
: Software that makes requests for one user at a time, either on the user's
  device or as a remote agent within a tenant's workload on a platform.

Bot:
: An agent with no user behind it, such as a crawler or a monitor, or a
  workload made of such agents. Having no user to shield, a bot identifies
  rather than being moderated anonymously.

Anchor:
: The MOLE role that grants Endorsements to Clients {{MOLE-ARCH}}.

Moderator:
: The MOLE role that issues Credentials and evaluates their presentations
  {{MOLE-ARCH}}.

Client:
: The MOLE role that redeems an Endorsement and presents the resulting
  Credential {{MOLE-ARCH}}. Which party plays it depends on the layer being
  moderated: a user's agent for MOLE(U), a workload or a component presenting
  on behalf of its agents for MOLE(W), a tenant for MOLE(T). This document
  uses the term only for that role.

The parties above form a hierarchy of layers. {{role-hierarchy}} shows the
delineation. Anchors are not in it: an Anchor is a role that any party with
a relationship to a layer can play.

~~~aasvg
Platform P
  |
  +-- Tenant T
  |     |
  |     +-- Workload W
  |     |     |
  |     |     +-- Agent A, acting for User U
  |     |
  |     +-- Bot workload B
  |
  +-- Intermediary
~~~
{: #role-hierarchy title="Layers of the hierarchy"}

This document also uses the following notation throughout.

Anchor X:
: An Anchor that endorses parties at layer X. For example, Anchor T endorses
  tenants, Anchor W endorses workloads, and so on.

MOLE(X):
: A MOLE Credential (or a presentation of one) derived from an Endorsement by
  Anchor X. It lets an origin moderate one party at layer X among all the
  parties at that layer that share Anchor X, without distinguishing them.
  For example, MOLE(T) is a Credential over all tenants endorsed by Anchor T.

WBA(X):
: A {{WBA-HTTPSIG}} signature by entity X, such as a platform, over a
  request. It lets an origin attribute the request to X. A tenant or
  workload that signs its own requests, for instance with the HTTP message
  signature profile of {{WIMSE-AIMS}}, adds a second labelled signature
  beside the platform's; WBA(T) or WBA(W) denotes it.

# Principles {#principles}

Finer-grained identity can give origins a finer-grained policy key, but it
also introduces four problems.

Security:
: An agent can spoof an unauthenticated identifier. Authenticating it
  establishes who supplied a request, not whether that request is harmless.

Equitability and scarcity:
: Origins need scarce budgets to keep agents from evading limits by starting
  over. Access to a trusted identity or an Anchor relationship is uneven,
  however. Scarcity based on those relationships cannot offer everyone the
  same opportunity to obtain a budget.

Privacy:
: A stable identifier lets origins correlate activity across requests,
  agents, and sessions, even when all they need is a limit on consumption.

Agency:
: Users should choose how much their agents disclose, from nothing through
  an anonymous Credential to a full identity ({{system}}). Requiring
  disclosure for any access to public content makes that choice a condition
  of participation.

The design goal for public resources is therefore a baseline level of access
without an identity, a platform signature, or a MOLE presentation. This floor
may be heavily rate limited and remains subject to available capacity and
request behavior. Scarce endorsements can support access above that floor.
Here, zero disclosure means no additional authentication signal. Ordinary
HTTP and network metadata remain visible.

This follows the guidance in {{Section 3.4 of CIVIL}}: "Absence of an active
signal does not itself establish abuse." Anonymous budgets and platform
signatures provide active signals for graduated enforcement while preserving
that baseline access.

Privacy partitioning {{PRIVACY-PARTITIONING}} provides the framing: separate
knowledge of who a user is from knowledge of what they do. An Anchor can use
its relationship with a user to endorse an agent, while an origin evaluates
consumption without learning that identity from the presentation. A provider
that also forwards requests may see both. This separation is a privacy goal,
not a guarantee against that provider or its collusion with the origin.

Moderation granularity should follow differences in behavior. Shared fate
within a workload may be acceptable while its agents behave alike. When one
user's behavior causes unrelated users to be throttled, the noisy neighbour
problem remains, even if its scope is smaller than a whole platform. Platforms
and workloads can help origins moderate those smaller units without requiring
stable identifiers for each of them. How to compose these limits remains an
open question.

Altogether, this leads to the following principles:

1. A platform can identify itself to origins, using {{WBA-HTTPSIG}}, rather
   than leaving origins to infer a platform from IP addresses. Bots and bot
   workloads on the platform identify under that identity or under a
   per-tenant key. Bots elsewhere identify with their own {{WBA-HTTPSIG}}
   identity.
2. A platform helps origins moderate its tenants and their workloads. It is
   the natural Anchor for them, endorsing each tenant, each workload whose
   agents act for users, or both; where another Anchor endorses them
   instead, the platform lets their presentations through. The endorsed
   party exchanges its Endorsements for Credentials that the origin
   moderates. A tenant identifies itself or its workload only if asked, and
   with consent of the tenant.
3. A workload helps origins moderate its individual users. The user's agent
   presents a Credential derived from an Endorsement by the user's Anchor,
   and the workload lets that presentation reach the origin. The user's
   Anchor may be the tenant, but need not be. A user identifies themselves
   only if asked, and with consent.

Implementing these principles requires no new protocol mechanism. MOLE
{{MOLE-ARCH}} and Web Bot Auth {{WBA-HTTPSIG}} exist and compose to
authenticate a request with graduated disclosure of identifying information.

# Attribution and authorization {#system}

Authorization, in this document, is an origin's decision to serve a request
and on what terms. An origin asks two questions to make this determination:
attribution (who, if anyone, can be held accountable for this request or
linked to others) and authority (what evidence permits the request and how
much may it consume).

Neither requires the other: a capability can prove a right without identifying
its holder, and a signature can identify a sender without granting it a right.
A session provides attribution across requests without necessarily naming
a person; an anonymous budget provides authority to consume without
identifying its holder. {{system}} describes the options.

## Attribution

An origin may attribute a request to a carrier, a workload, or a user. A
WBA(P) {{WBA-HTTPSIG}} signature identifies the platform or bot that carried
it. The origin can hold the signer to an aggregate limit or send it abuse
reports without learning who the agent, tenant, or user behind it is. A
workload identity in the sense of {{WIMSE-AIMS}} identifies which software
is acting, stably across its requests, without identifying its users. A
WIMSE identifier is bound to a short-lived credential and proven by mTLS,
a Workload-Proof-Token, or an HTTP message signature, written WBA(W) or
WBA(T) here. A login or subject-bearing access token can also attribute a
request to a person. An origin can ask for a signature using
Accept-Signature {{HTTPSIG}} or for workload authentication using its
scheme; a platform signature is offered opportunistically and cannot be
compelled. A signature vouches for its signer, but grants no right unless
the resource's policy grants one to that signer.

A session identifier, as in {{POPPY}} or {{AAP}}, provides attribution
across requests without necessarily identifying anyone. It can be added to
any of the evidence above, letting the origin see a task unfold. It is
established by whatever scheme the origin uses for sessions. Attribution
may thus be absent, per session, or per principal. Per-principal attribution
lets an origin build a profile; even an artifact that names no one but is
reused across requests, such as a bearer capability, is linkable per
artifact. With no linkability, the origin decides on what each request
carries; MOLE carries spending history in a hidden balance rather than
linking presentations.

## Authority

Authority to act can be proved without attributing the request to a person.
A capability proves that its holder has a right, such as read or write
access to an API, without identifying the holder or requiring their consent
at the time of use. The issuer must be trusted by the origin. Privacy Pass
{{PRIVACYPASS}} and its PrivateToken scheme {{PRIVATETOKEN}} offer unlinkable
proof of an issuer's attestation, where that attestation suffices for the
origin's access decision. Bearer capabilities with attenuated rights or
anonymous credentials that prove an attribute can convey other rights
without a principal. Capabilities are asked for with their scheme's
challenge.

When the origin needs permissions tied to a principal, a workload can obtain
an access token for itself: its platform's Workload Identity Provider signs
an assertion naming it by an opaque, never-reassigned identifier, optionally
with scopes and claims such as a tenant, and the origin's authorization
server exchanges it for a token {{WAG}}. An OAuth access token {{OAUTH}}
with a human subject can instead convey permissions delegated through the
authorization code grant, {{OAUTH-XAA}}, or {{AAUTH}}, with the agent as the
token's client. An authenticated person can also use {{OIDC}} or the
origin's own account. Tokens are asked for with a Bearer challenge
{{BEARER}} pointing to protected resource metadata {{OAUTH-RS-META}}, and
accounts with the origin's login. If a token is present but insufficient,
{{STEPUP}} names what is missing. The right required by a resource, rather
than a preference for identifying a principal, determines which evidence
the origin asks for.

Capacity is authority to consume, distinct from authority to act. MOLE(X)
presents a Credential at tenant, workload, or user grain. The origin learns
that one member of Anchor X's set has budget left, and the balance carries
that member's history with this origin into later requests.
It cannot tell which member, nor link two presentations. The origin asks
for this authority with the Mole challenge of {{MOLE-HTTP}}. A MOLE budget
answers how much, not what actions are permitted; a capability proving a
right to act does not supply a spendable balance.

The origin's trust decision depends on who vouches for each kind of
authority: an Anchor hidden in an Anchor Set it curates, a capability
issuer whose keys it accepts, a Workload Identity Provider registration it
can revoke, or an authorization server or identity provider it federates
with. These differ in recourse and in what the origin learns about the
requester.

## Composition

Attribution and different forms of authority can compose on one request.
Platform and workload signatures can occupy distinct labels in the Signature
fields; a workload may instead use a Workload-Proof-Token. Workload and human
authority compose inside one access token, as {{WIMSE-AIMS}} describes, e.g.,
where the workload is the token's client and the person its subject. Capacity
composes with a WBA signature or with a capability: an identified workload may
have anonymous users with separate budgets, or a capability holder may prove
both a right and a remaining budget. The stacked example of {{examples}}
illustrates the former arrangement.

Capacity with a subject-bearing token rarely needs to compose. A token with
a human subject already gives the origin a key to meter on, and adding
MOLE(U) beside it protects nothing the token has not disclosed; a workload
with scoped permissions of its own can carry its identity as a signature.
Capacity with a capability is different. Both are anonymous, and a request
that proves it may write and that it has budget left is a natural
combination. Today it has no encoding: PrivateToken and MOLE both occupy the
Authorization field, which holds one credentials value
({{Section 11.6.2 of HTTP}}). This is the case that motivates a field other
than Authorization for presentations in {{MOLE-HTTP}}, which this document
does not define.

An origin that needs several kinds of evidence asks for several. A
WWW-Authenticate field may carry several challenges ({{Section 11.6.1 of
HTTP}}). How an origin signals that it requires all of them rather than any
one remains open. {{processing-steps}} describes the exchange.

# Moderating access {#processing-steps}

In practice, agentic stacks differ in many ways:

- how many users, agents, workloads, tenants, and platforms there are, and
  how they are arranged;
- who signs, who endorses, and how many intermediaries sit on the path; and
- whether the agent runs on the user's device or on a platform.

By default, none of this is known to the origin that is moderating access to
its resources, and none of it needs to be. The origin sees a request that
discloses some of the options of {{system}}. It can moderate on that basis, or
challenge for more.

This is what lets one implementation serve many scenarios. An origin does not
configure itself per platform or per topology. It configures a Moderator
policy, including an Anchor Set, one or more credential contexts, and the
amounts it charges and refunds, together with the authority schemes its
permissioned resources use. Everything else follows from the exchange on
each request: the agent offers what it is willing to disclose, the origin
accepts the offer if it is enough for what this request needs, and otherwise
challenges for more. What it challenges for depends on the need, not on the
topology behind the request. The protocol exchanges and HTTP carriage for the
anonymous budget are defined in {{MOLE-PROTOCOLS}} and {{MOLE-HTTP}}; the
subsections below describe the exchange, and {{flow1}} walks through one.

## Request offer {#offers}

A request can offer attribution, authority, both, or neither. A bare request
offers neither; on a public resource it receives the baseline, possibly
heavily rate limited. An intermediary may add its own attribution as the
request passes through it (with a {{WBA-HTTPSIG}} signature) without changing
what the agent chose to disclose about itself.

In general, clients can opportunistically offer some attribution or authorization
information, should they be configured to do so by the application or some other
policy. However, for the most part, requests offer minimal information, especially
when they have no a priori arrangement about who they are interacting with.

## Offer acceptance {#accepting}

The origin verifies the offered evidence before deciding what to attribute
and what authority to accept. An invalid signal does not establish the
claim it was offered for; the origin evaluates whatever other evidence is
valid.

The offer is valid when its accepted attribution and authority meet the
request class's policy. For most public content, the origin needs only to
manage consumption and keep users from affecting one another; a valid MOLE
presentation at the grain it cares about can suffice. If the offer falls
short, the origin serves the public baseline, which may be very little,
and challenges for what is missing. A permissioned resource can refuse
access until its required authority is provided.

### Attribution

The origin verifies each platform or workload signature or proof token
against its signer's credential. A valid platform signature attributes the
request to its signer and places it under that signer's aggregate limit. An
absent or invalid signature provides no attribution to that signer; the
origin can still apply its address-based limit or use another verified
signal. An absent signature is not a failure: an origin can ask for one
with Accept-Signature {{HTTPSIG}} but cannot count on receiving it. A
session or login is verified by the origin's own means; a verified
subject-bearing token also attributes the request to its subject. When a
signature and MOLE presentation arrive together, the signature is expected
to cover the field carrying the presentation, so that the signer is
accountable for the pairing; see {{security}}.

### Authority

For authority to consume, the origin handles redemption and presentation
separately. A CredentialRequest answers a redemption Challenge: the origin
verifies the Redemption against the Anchor Set, checks that the
Endorsement's nullifier has not been redeemed, chooses an initial balance
from policy, records the nullifier, and returns a Credential rather than
content. A presentation answers a presentation Challenge: the origin
determines the Challenge from the credential context and request, confirms
that the key and context are current, rejects a replayed nullifier, checks
that spend and top-up match the Challenge, and verifies the proof and
predicate that the balance covers the spend. A failed presentation supplies
no authority to consume. Whenever an intermediary is on the path, the
Moderator uses the request-bound profile of {{MOLE-PROTOCOLS}}, with a
request context covering the method, target, and body, so that a
presentation seen in transit cannot be attached to a different request;
see {{security}}. Neither exchange identifies the member of the Anchor Set.

For authority to act, the origin verifies a capability against its issuer's
key, checks the right it conveys, and, for single-use schemes, checks its
redemption state. A capability need not identify its holder. An access
token is verified as the origin's authorization server requires: issuer,
audience, expiry, and scope, and, when sender-constrained with {{DPOP}} or
a client certificate, proof that the sender holds the key. The origin
checks that the resulting permissions cover the requested action. A login
is verified by the origin's own means, and any right based on workload
identity depends on the resource's policy for that identity. A verified
identity alone does not imply permission to act.

## Offer challenge {#challenging}

The origin challenges for the missing attribution, authority to consume,
or authority to act that its policy needs. Asking for a right does not
require asking for an identity.

The challenge is interactive. The origin asks in response to a specific
request, and the agent answers on the request that follows. Nothing is
published in advance, and the origin chooses when, and of whom, to ask. An
origin that sees nothing of concern may never challenge, whereas one
under load may challenge every request.

### Attribution

If the origin needs to know who carried or made the request, for instance
to apply an aggregate limit to a platform or restrict a resource to known
workloads, it can ask for a signature with Accept-Signature {{HTTPSIG}} or
use the workload's authentication scheme. A platform cannot be compelled
to sign, and a direct agent may have no platform signature to offer. Where
the origin needs session-level attribution, it asks for a session under
its own scheme. A signature establishes no permission unless the resource
policy grants a right to that signer.

### Authority

To limit consumption per workload or user without knowing who they are,
the origin challenges for MOLE. The origin challenges at the level of
granularity it wants: tenant, workload, user, or several, and may offer several
endorsement or credential types at once; the agent ignores types it does
not support and answers one it does. (Challenge granularity depends on
having anchor sets that correspond to that granularity, e.g., challenging
for a user credential requires an anchor set that endorses users as opposed
to workloads.)

To authorize an action on a permissioned resource, such as read or write
access to an API, the origin asks for evidence of a right. Where a right
suffices without identifying its holder, it issues the capability scheme's
challenge, such as a PrivateToken challenge {{PRIVATETOKEN}} if the issuer's
attestation conveys the required right. Where the decision depends on a
principal's permissions, it issues a Bearer challenge {{BEARER}} naming the
resource metadata {{OAUTH-RS-META}} from which the agent learns the
authorization server. A workload acting for itself obtains its token there
with a {{WAG}} assertion, and an agent acting for a person through
{{OAUTH-XAA}} or {{AAUTH}}. Where a token is present but too weak, {{STEPUP}}
names what is missing. For a resource whose policy grants access to known
workloads, workload authentication can itself be sufficient evidence. For
an account-bound action, the origin may require login. MOLE alone does not
establish a right to act, and the origin should not infer one from a
presentation.

The origin can request both kinds of authority when it needs a right to
act and an anonymous consumption limit. With a token bearing a human
subject, it can instead meter on that subject without also asking for
MOLE(U). To moderate anonymous users behind an identified workload, it can
ask for a workload signature and MOLE(U). It should ask for no more than the
request needs, and the agent is free to decline any challenge. A public
resource keeps serving the baseline, since that floor is what makes
disclosure voluntary rather than coerced. A permissioned resource has no
public floor and may refuse.

## Moderation {#serving}

Having accepted or challenged, the origin serves the request at the level
the accepted offer warrants and, where applicable, updates whatever quota
or capacity is allocated for the requesting client and its session.
(With MOLE, this is via a credential update, whereas with an identified user,
this might be with per-user rate limits.)

An origin shapes a client session in a few ways.

* Pricing by request. Different request classes carry different spend
  amounts, typically set by the challenge the request answers.
* Rewarding behaviour. A policy may refund a request that completes normally
  in full, so the agent keeps its balance. A request that trips an abuse
  signal or fails authorization may receive a partial or zero refund.
  The balance then reflects those decisions without identifying the agent.
* Topping up. A top-up a above zero lets the origin grow the balance of an
  agent it has come to trust, bootstrapping access from a cautious initial
  balance, and so accepting Anchors it trusts less.
* Refreshing. An offer that does not increment or decrement the client's quota.
* Ending the session. Consuming a request and then invalidating their remaining
  quota, either by way of dropping or invalidating state (e.g., in MOLE, without
  returning an update and invalidating the client's credential).

A policy that fully refunds successful requests does not bound their total
consumption, since an agent can repeat them without depleting its balance.
These policies describe behavior-based spending, not a per-user rate limit,
though per-user limiting is possible using per-user identifying information
(such as their ID conveyed in a token).

\[\[ Editor's note: there might be explicit sessions with delayed Update;
this needs discussion. \]\]

## Request flow: WBA + MOLE {#flow1}

{{sequence}} shows the exchange of {{processing-steps}} for an origin that
needs a fair share and an agent that offers nothing, behind an intermediary
that adds its platform's signature, as a request moves from a remote agent
through that intermediary to an origin's Moderator. The exchanges follow
{{MOLE-PROTOCOLS}} (the message names below are those of that document).

~~~
Anchor       Agent      Intermediary (P)       Origin
  |            |                   |              (Moderator)
  | Endorsement|                   |                 |
  |----------->|                   |                 |
  |            |                   |                 |
  |            | (1) request, no presentation        |
  |            |------------------>| add WBA(P)      |
  |            |                   |---------------->| verify WBA(P),
  |            |                   |                 | aggregate limit
  |            | (2) baseline response + Challenges  |
  |            |<------------------|<----------------|
  |            |   redemption: redemption_context    |
  |            |   presentation: spend s, top-up a   |
  |            |                   |                 |
  |            | (3) CredentialRequest               |
  |            |------------------>|---------------->| verify Redemption
  |            |   Redemption over Anchor Set        | over Anchor Set,
  |            |   + IssuanceRequest                 | check nullifier,
  |            |                   |                 | balance c, record
  |            | (4) CredentialResponse              |
  |            |<------------------|<----------------|
  |            | Credential (c)    |                 |
  |            |                   |                 |
  |            | (5) request + Presentation(s, a)    |
  |            |------------------>| add WBA(P)      |
  |            |                   |---------------->| verify WBA(P),
  |            |                   |                 | aggregate limit;
  |            |                   |                 | nullifier lookup;
  |            |                   |                 | VerifySpend:
  |            |                   |                 |  proof, c >= s;
  |            |                   |                 | choose refund t;
  |            |                   |                 | record; serve
  |            | (6) response + Update + Challenge   |
  |            |<------------------|<----------------|
  |            | Credential (c-s+t)|                 |
  |            |                   |                 |
  |            | (7) repeat from (5), answering each Challenge
~~~
{: #sequence title="A request moving through the system"}

1. The agent sends a request with no presentation. The intermediary adds the
   platform's {{WBA-HTTPSIG}} signature and forwards it.
2. The origin verifies the signature, if there is one, attributes the request
   to the platform, and applies its aggregate limit. Finding no presentation,
   and wanting this request class moderated, it serves the baseline level of
   access, possibly heavily rate limited, and issues both Challenges. The
   redemption Challenge carries a redemption context that names the Moderator,
   its policy, and a time window. The presentation Challenge carries the
   credential context and the terms of a presentation: a spend amount s and
   a maximum top-up a. Because the baseline was served, the Challenges travel
   on a 200 response and are optional.
3. The agent has no Credential for this Moderator, so it answers the
   redemption Challenge first. It fetches the Moderator's configuration from
   the Redeem and Issue endpoint, which lists the Anchor Set, and sends a
   CredentialRequest to that endpoint: a Redemption that proves its
   Endorsement comes from some Anchor in the set, without revealing which,
   and an IssuanceRequest for a Credential.
4. The origin verifies the Redemption, checks that the Endorsement's
   nullifier has not been redeemed before, chooses an initial balance c
   according to its policy, records the nullifier, and returns a
   CredentialResponse. The agent finalizes it into a Credential with hidden
   balance c.
5. The agent now answers the presentation Challenge. It repeats the request
   with a Presentation that proves its balance covers the spend amount s
   without revealing the balance, and includes a request for a top-up of at
   most a. The intermediary signs as before, if there is one.
6. The origin verifies the signature, if any, and applies the aggregate
   limit as in step 2. It then looks up the presentation's nullifier to reject a replay,
   verifies the proof and the predicate that the balance is at least s,
   chooses a refund t between 0 and s + a, records the result under the
   nullifier, and serves the request. The response carries an Update with
   the refund, and a new presentation Challenge for the agent's next
   request. The agent finalizes the Update into a fresh Credential with
   balance c - s + t.
7. Subsequent requests repeat from step 5, each answering the Challenge
   carried on the previous response. Each presentation uses a fresh
   nullifier, so the origin cannot link one request to the next.

Every presentation answers a Challenge. An agent never presents unprompted,
and a presentation that answers no Challenge the origin issued is invalid.
For the chain to continue, each response that the origin wants followed by a
moderated request carries the next presentation Challenge, as step 6 shows.
An origin that stops challenging stops receiving presentations, and the agent
falls back to the baseline. The Challenge is the origin's to issue, when and
to whom it chooses, in response to a request it has seen. Graduated
disclosure, as described in {{principles}}, is the origin's choice of when to
ask.

If the presentation fails verification, or the balance does not cover s, the
origin falls back to the baseline response of step 2. The agent can obtain a
new Credential only by redeeming another Endorsement, which its Anchor grants
scarcely. That is what makes the limit bind.

# Deployment examples {#examples}

The following examples separate the traffic path from the endorsement
relationship, using the notation of {{terminology}}. An Anchor may be the
platform, a tenant, or an independent party accepted by the origin. The
origin's Moderator may be operated by the origin or delegated to a provider
acting on its behalf. A platform signature allows a separate aggregate limit.
Signing forwarded requests needs the profile discussed below. A tunnel
carrying the agent's TLS session untouched cannot add such a signature.

In {{fig-intermediary}}, a device agent presents MOLE(U) through the HTTP
intermediary of platform P, which adds WBA(P). Anchor U is drawn separately
to show that it need not operate that intermediary. A consumer AI service can
combine these roles and host the agent.

~~~aasvg
+----------+
| Anchor U |
+----+-----+
     | Endorsement
     v
+----------+           +--------------+          +-----------+
|  Device  +-MOLE(U)-->| Intermediary +-MOLE(U)->|   Origin  |
|  agent   |           | (platform P) |  +WBA(P) | Moderator |
+----------+           +--------------+          +-----------+
~~~
{: #fig-intermediary title="A device agent reaching an origin through platform P, endorsed by Anchor U"}

In {{fig-direct}}, the agent connects directly. It still has a relationship
with Anchor U and presents MOLE(U) without any signature on the request.

~~~aasvg
+----------+
| Anchor U |
+----+-----+
     | Endorsement
     v
+----------+                                    +-----------+
|  Device  +-------------- MOLE(U) ------------>|   Origin  |
|  agent   |             no WBA(X)              | Moderator |
+----------+                                    +-----------+
~~~
{: #fig-direct title="A direct connection with Anchor U off the traffic path"}

Anchor W, typically platform P itself, can endorse a hosted workload, and
Anchor U can endorse its users' agents. {{fig-stacked}} illustrates the
intended scopes of a stacked arrangement: the request carries MOLE(W) and
MOLE(U), which are workload and user budgets, not visible identifiers, and
WBA(P). How several presentations compose on one request, and any shared-budget
mechanism, are not yet defined; see the editor's note below.

~~~aasvg
+---------------------+       +---------------------+
|      Anchor W       |       |      Anchor U       |
+----------+----------+       +----------+----------+
           | Endorsement                 | Endorsement
           v                             v
+--------------------------------------------------+
| Platform P                                       |
|  +--------------------------------------------+  |
|  | Tenant workload with user agents           |  |
|  +----------------------+---------------------+  |
|                         |                        |
|                         v                        |
|                 +--------------+                 |
|                 | Intermediary +-----------------+---+
|                 +--------------+                 |   |
+--------------------------------------------------+   |
                                                       | MOLE(W)
                                                       | MOLE(U)
                                                       | +WBA(P)
                                                       v
                                                 +-----------+
                                                 |   Origin  |
                                                 | Moderator |
                                                 +-----------+
~~~
{: #fig-stacked title="Workload and user budget scopes in a proposed stacked deployment"}

An origin may delegate the Moderator to a CDN or similar service that fronts
it. Any of the examples above can sit in front of it. The Moderator's Anchor
Set and policy are then shared across the origins the CDN serves under that
configuration, which makes the anonymity set larger. A Credential issued by
the CDN is valid at every origin it moderates under the same context. The CDN
sees presentations across those origins. MOLE keeps them unlinkable, but
source addresses and timing remain visible to it (see {{security}}).

A request may also cross more than one intermediary, for instance a device
agent behind an enterprise ZTNA gateway whose traffic then leaves through a
consumer VPN. Only an intermediary that terminates HTTP can sign. The origin
can attribute the request to every intermediary that signs, and sees a tunnel
only as a source address. Aggregate limits apply per signer. The MOLE
presentations are unaffected, because the Anchors that endorsed the workload
and the user are not tied to any intermediary on the path.

\[\[ Editor's note: Open points include:

* Budget allocation: how to apply limits to users and individual agents,
  including a user with several agents or a workload with many users.
* Concurrent requests: one Credential chain cannot be used in parallel.
  Several Credentials provide separate budgets. Splitting a shared budget
  and enforcing workload and user limits together need further work. One
  option is for an Anchor to grant a workload k Endorsements per epoch,
  redeemed into a pool of k Credentials, one per concurrent agent. How to
  choose k, and whether the origin should see it, is open.
* Intermediary signing: signing forwarded requests on behalf of others may
  require a profile of {{WBA-HTTPSIG}}, including its binding to MOLE
  presentations (see {{security}}).
* Chained intermediaries: when several HTTP-terminating intermediaries sign,
  which signers' aggregate limits the origin should apply.
* Capacity beside a capability: a request proving a right and a budget at
  once is a natural combination, but PrivateToken and MOLE both occupy the
  Authorization field. {{MOLE-HTTP}} would need a field other than
  Authorization for presentations; see {{system}}.
  \]\]

# Security Considerations {#security}

The security of MOLE itself is treated in {{MOLE-ARCH}} and
{{MOLE-PROTOCOLS}}.

## Provider collusion and metadata

Anonymous presentations do not prevent a forwarding
provider that knows the user from disclosing which requests are theirs to the
origin. The deployment does not protect against this provider-origin collusion.

Unlinkable presentations do not hide source addresses, cookies, request
contents, or timing. A platform signature also narrows the anonymity set.
Origins must not infer that issuer hiding conceals an intermediary already
identified by its signature or address.

## Anchor issuance and trust

An Anchor controls how many Endorsements exist. The origin's recourse against
one that does not constrain issuance is to drop it from the Anchor Set.
Credential lifetime and retirement determine how long existing budgets remain
usable. Endorsement reuse and replay protection depend on the protocol chosen.

## Presentation reuse by intermediaries

An HTTP intermediary that can read a presentation in transit may try to use it
on a request of its own before the agent's request arrives. Single-use
verification prevents two accepted uses, but does not decide which request is
authorized. This deployment therefore needs request-bound presentations that
cover all request components affecting authorization, including the body where
relevant. Request binding does not prevent an intermediary from racing an
identical request or withholding it.

## Access tokens crossing intermediaries

The same intermediary can read a bearer access token in transit and replay
it on a request of its own, with none of the single-use protection a
presentation has. Where human or workload authority travels through an
intermediary, the token should be sender-constrained, with {{DPOP}} or a
client certificate, as {{WIMSE-AIMS}} recommends. That binds the token to a
key the intermediary does not hold, as request binding ties a presentation
to its request.

## Binding signatures to presentations

A composition profile should also require the Web Bot Auth signature to cover
the field carrying the MOLE presentation, along with the relevant request
components. This binds the signer's attribution to that presentation. It does
not replace request binding against a malicious signer. This document does
not define the profile or claim that existing signatures provide this binding.

## Multiple budgets across Anchors

A user or workload with Endorsements from several accepted Anchors can obtain
several independent budgets. Unlinkable presentations do not let the origin
detect that those budgets belong to the same party. Anchor Set selection and
issuance policy affect the exposure but do not enforce one budget per user
across Anchors.

## Unsigned requests

An absent signature is not evidence of an attack. Direct agents and agents
behind tunnels may have no platform signature to offer. For public resources,
origins should preserve the baseline access described in {{principles}}, while
applying capacity and behavior-based controls.

# IANA Considerations {#iana}

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
