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
  MTLS: RFC8705
  DIGEST: RFC9530
  EC-ACT: I-D.draft-schlesinger-mole-ec-act
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

Origins often limit requests by address or by the platform carrying them.
Clients behind shared infrastructure then share the consequences of one
another's activity. This document sketches independently allocated budgets
that let origins bound consumption without requiring stable Client identifiers.
It describes how MOLE credentials preserve budget continuity across unlinkable
presentations, how carrier attribution supports aggregate limits, and where
composition with resource authorization needs further work.

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

This document uses the HTTP terminology of {{HTTP}} and the MOLE terminology
of {{MOLE-ARCH}}. The roles can be operated by separate parties or combined.

Client:
: The MOLE participant that obtains an Endorsement, redeems it for a
  Credential, and presents that Credential when requesting a resource.

Anchor:
: The party that grants Endorsements under a policy the Moderator accepts.
  Its relationship with a Client constrains access to new budgets.

Moderator:
: The party that issues Credentials and verifies their presentations. It
  sets the budget policy and may be operated by the origin or on its behalf.

Origin:
: The server that controls access to the requested resource.

Intermediary:
: A proxy, gateway, or tunnel on the request path. An intermediary that
  terminates HTTP can sign forwarded requests under its own identity.

A Client may hold several budgets, and an account or workload may receive
budgets for several Clients. Those allocations depend on Anchor and Moderator
policy. A workload has the meaning given in {{WIMSE-ARCH}}. A Credential does
not establish a unique account, workload, device, or other principal.

MOLE(C) denotes a presentation from Client C. WBA(P) denotes a Web Bot Auth
signature from intermediary P. C is a label in the figures, not an identifier
sent to the origin. A credential chain is the sequence of Credentials obtained
by updating one initial Credential. The Client experiences continuity through
that chain without giving the origin a session identifier.

# Scope and principles {#principles}

The aim is to bound consumption through independently allocated budgets
without requiring a stable Client identifier. Exhausting one budget should
not exhaust an unrelated budget. Enforcement still depends on constrained
issuance and available capacity. A platform aggregate limit can affect every
Client behind it even when their individual budgets remain usable.

For public resources, the deployment goal is a baseline level of access
without a Credential or signature. This baseline remains subject to capacity
and request behavior. Scarce Endorsements can support access above it.
Absence of an active signal does not itself establish abuse {{CIVIL}}.
Permissioned resources may require authorization before serving any content.

Clients choose which evidence to offer. Origins should ask only for evidence
needed by the requested operation. A verified identity does not establish that
a request is harmless, and a valid budget does not grant every permission.
Access to accepted Anchors is uneven, so scarce issuance alone does not ensure
fair allocation.

The privacy objective is to separate the relationship used to obtain a budget
from the requests that spend it {{PRIVACY-PARTITIONING}}. Presentations should
reveal only what the origin needs to apply its policy. Network metadata and
other request fields can still identify or correlate Clients.

The flows below use the EC-ACT credential type of {{MOLE-PROTOCOLS}}. Other
MOLE credential types can have different properties. This is a deployment
sketch using work in progress. The composition gaps are listed in
{{composition}}, and the underlying construction's security analysis remains
incomplete {{EC-ACT}}.

# Attribution and authorization {#system}

An origin decides whether to serve an operation and how much capacity to
allocate to it. The evidence it accepts can establish attribution, permission,
or a remaining budget. A request may need more than one.

## Attribution

A verified WBA(P) signature attributes its covered components to a key
published by P {{WBA-HTTPSIG}}. The origin can apply an aggregate limit to P
without receiving an identifier for each Client behind it. The signature does
not establish permission unless origin policy grants it to that signer.

A workload can authenticate using a scheme such as {{WIMSE-AIMS}}. Its
credential and signature profile differ from Web Bot Auth, even when both
use HTTP Message Signatures {{HTTPSIG}}. A session identifier allows requests
to be correlated within the session. Either form of attribution can accompany
a budget, with the resulting loss of unlinkability within its scope.

An origin can request a signature with Accept-Signature. The Client may decline,
and the origin may refuse access where its policy requires a signature.
The public baseline in {{principles}} is this deployment's goal, not a
requirement imposed by Web Bot Auth.

## Permission

Permission depends on the resource's policy. An origin can accept a capability
without identifying its holder, including a Privacy Pass token when the
issuer's attestation satisfies that policy {{PRIVACYPASS}} {{PRIVATETOKEN}}.
A reused capability can still link requests.

An OAuth access token conveys an authorization grant {{OAUTH}}. The resource
checks the token as its authorization server requires, including its audience,
validity, permissions, and any proof of possession. Identity obtained through
{{OIDC}} or workload authentication can also be an input to access policy.
Authentication alone does not determine the permitted operations.

A workload can obtain a token using {{WAG}}, and an existing federation can
support delegated access through {{OAUTH-XAA}}. These grants are exchanged at
the authorization server, which decides what access token to issue. They do
not define a common token claim layout. {{AAUTH}} defines a separate signed
authorization protocol with its own challenges and credentials.

## Capacity

An EC-ACT Credential holds a hidden balance c. A presentation proves that it
covers a requested spend s. The Moderator can return an amount t, bounded by
s plus the offered top-up a. The replacement Credential holds c - s + t.
The origin learns the public amounts and that the spend was possible, without
learning the balance or a stable Client identifier {{MOLE-PROTOCOLS}}.

For example, a policy can issue B credits, charge one per accepted presentation,
and return zero with no top-ups. Each issuance then allows at most B fresh
accepted operation attempts. Further issuances increase that allocation.
Bounding consumption over time also requires policy for replenishment and
expiry. Refunding every spend in full leaves successful consumption unbounded.

A principal that already authenticates may still need several independent
budgets. Metering that principal does not replace those allocations. Conversely,
adding MOLE to requests that already carry a stable identifier does not make
those requests unlinkable.

## Composition {#composition}

A carrier signature and a MOLE presentation can travel on one request. The
signature should cover the presentation and the relevant request components,
as discussed in {{security}}. Multiple signature profiles can use distinct
labels, with each verifier applying the corresponding profile.

Permission and capacity can also be required together. However, MOLE,
PrivateToken, and OAuth access tokens use the Authorization field, which
carries one credentials value {{Section 11.6.2 of HTTP}}. The current MOLE
transport also selects one presentation per request {{MOLE-HTTP}}. Carriage
of several budgets or of a budget beside another authorization scheme remains
undefined here.

WWW-Authenticate can carry several challenges, but that alone does not express
that all must be satisfied. Signaling cumulative requirements is also open.
The permissioned flow in {{flow2}} marks this boundary rather than specifying
an encoding.

# Moderating consumption {#processing-steps}

The origin configures its accepted Anchor Set, credential contexts, allocation
and spending policy, and any permission requirements. A Client may send an
initial request without evidence. The origin verifies what is present and
serves the level of access its policy permits. A public response can offer
more capacity through a MOLE challenge. A permissioned resource can refuse the
operation and challenge for the missing authorization.

A MOLE redemption challenge lets a Client obtain a Credential. A presentation
challenge specifies its spending terms. Clients use the exchanges and
verification rules of {{MOLE-PROTOCOLS}} and {{MOLE-HTTP}}, including replay
checks and request binding. A failed proof supplies no evidence of budget.

For EC-ACT, the Moderator records the spend result atomically before attempting
the operation. It chooses a refund or NoUpdate from the evidence available at
that point. The response carries that recorded result even if the operation
fails. Choosing a refund from the operation's eventual outcome would require
additional settlement behavior not defined by this flow.

A refund replaces the spent Credential. A zero refund leaves c - s credits.
NoUpdate ends the chain and forfeits its remaining balance. At the HTTP layer,
an empty Mole-Credential update conveys NoUpdate. A missing update does not.

After a lost response, the Client can recover the recorded credential result
within the retention period. Recovery does not repeat the protected operation
or establish whether it completed. Application recovery remains separate.
One credential chain permits one presentation in flight at a time.

## Public access with WBA and MOLE {#flow1}

The origin offers a budget above its public baseline. Client C already holds
an Endorsement from an accepted Anchor. Intermediary P adds its own signature.
The Moderator is shown at the origin.

~~~
Anchor        Client C      Intermediary P      Origin
  |               |                |          (Moderator)
  | Endorsement   |                |               |
  |-------------->|                |               |
  |               | request        |               |
  |               |--------------->| + WBA(P)       |
  |               |                |-------------->|
  |               | baseline + redemption and      |
  |               | presentation challenges        |
  |               |<---------------|<--------------|
  |               |                |               |
  |               | Redeem & Issue |               |
  |               |--------------->|-------------->|
  |               | Credential with balance c      |
  |               |<---------------|<--------------|
  |               |                |               |
  |               | request + MOLE(C)              |
  |               |--------------->| + WBA(P)       |
  |               |                |-------------->|
  |               |                |     record spend result
  |               |                |     attempt operation
  |               | response + Update              |
  |               |<---------------|<--------------|
  |               | balance c - s + t              |
~~~
{: #sequence title="Public access with carrier attribution and an anonymous budget"}

The origin verifies WBA(P) and applies any aggregate limit separately from the
Client's budget. Because it serves the baseline, its optional MOLE challenges
travel on a 200 response. A Client with a suitable Credential can skip
redemption. Otherwise, it obtains the Moderator configuration and redeems its
Endorsement at the Redeem & Issue endpoint.

On the budgeted request, the Client proves that its balance covers s and
requests a top-up of at most a. The Moderator verifies and records the result
before attempting the operation. The example continues the chain by returning
an Update with 0 <= t <= s + a. Each subsequent spend answers an applicable
presentation challenge. Successive fresh presentations have different
nullifiers, and the Credential does not provide a link between them.

A direct Client uses the same exchange without WBA(P). A Client unable or
unwilling to present continues at the public baseline, subject to origin policy.
A failed presentation does not authorize the budgeted level of access.

## Permissioned access with a separate budget {#flow2}

The Client requests an operation requiring an OAuth access token. The resource
uses protected resource metadata {{OAUTH-RS-META}} to identify its authorization
server. The grant issuer below is a Workload Identity Provider for WAG, or an
Identity Provider Authorization Server for ID-JAG. Each requires an existing
trust arrangement with the resource's authorization server.

~~~
Grant issuer     Client       Authorization server     Resource
     |              |                  |                  |
     |              | operation request                   |
     |              |------------------------------------>|
     |              | challenge + resource metadata       |
     |              |<------------------------------------|
     |              |                  |                  |
     |<--- request --|                  |                  |
     |---- grant --->|                  |                  |
     |              | grant exchange   |                  |
     |              |----------------->|                  |
     |              | access token     |                  |
     |              |<-----------------|                  |
     |              |                  |                  |
     |              | operation + access token            |
     |              | + proof of possession, if required  |
     |              |------------------------------------>|
     |              |                  |     verify permission
     |              |                  |     apply capacity policy
     |              | response         |                  |
     |              |<------------------------------------|
     |              |                  |                  |
     |              | Optional separate MOLE budget       |
     |              | follows flow 1. Combined carriage   |
     |              | remains an open composition point.  |
~~~
{: #permission-sequence title="Permissioned access with optional independent budget enforcement"}

For WAG, the Client exchanges a workload assertion for an access token. For
ID-JAG, it obtains a grant through the existing federation and exchanges it
at the resource's authorization server. The resource receives the resulting
access token, not either grant. A Client with a suitable token skips this
exchange. Bearer challenges follow {{BEARER}}. DPoP uses its own authentication
scheme {{DPOP}}. Insufficient scope and authentication strength or freshness
are distinct failures, handled by {{BEARER}} and {{STEPUP}}, respectively.

The resource can meter an authenticated principal or require a separate budget
for the Client's activity. Several Clients authorized under the same principal
can have independent allocations. Requiring both permission and a MOLE budget
needs the composition work described in {{composition}}. A workload signature
alone does not replace the access token's permissions.

# Deployment considerations {#examples}

An endorsement relationship is independent of the traffic path. In
{{fig-intermediary}}, the Anchor endorses Client C while
intermediary P carries its requests. The same provider can operate both roles.
A tunnel carrying end-to-end TLS cannot add an HTTP signature.

~~~aasvg
+----------+
|  Anchor  |
+----+-----+
     | Endorsement
     v
+----------+           +--------------+           +-----------+
| Client C |  MOLE(C)  | Intermediary |  MOLE(C)  |   Origin  |
|          +---------->|      P       +---------->| Moderator |
|          |           |              |  +WBA(P)  |           |
+----------+           +--------------+           +-----------+
~~~
{: #fig-intermediary title="Separate endorsement and forwarding relationships"}

Independent allocations can coexist under one authenticated principal, as in
{{fig-budgets}}. Permission covers the operation each Client may request.
Budget policy determines how much each may consume. Issuance policy must
constrain the total allocation if creating another Client is cheap.

~~~aasvg
+-----------------------------+
| One authorized principal    |
|                             |
| +----------+  +----------+  |
| | Client A |  | Client B |  |
| | Budget A |  | Budget B |  |
| +-----+----+  +-----+----+  |
+-------+-------------+-------+
        |             |
        v             v
+-----------------------------+
| Resource                    |
| Permission for each request |
| Separate capacity budgets   |
+-----------------------------+
~~~
{: #fig-budgets title="Independent budgets under one principal, with combined carriage still undefined"}

Concurrent requests need separate credential chains. Several Credentials
provide separate balances, so assigning more Credentials can increase both
concurrency and total consumption. Shared-budget splitting and simultaneous
checks of aggregate and individual anonymous budgets remain open.

A CDN can operate a Moderator for several origins. All servers accepting the
same credential key and context must share atomic replay state. Independent
stores need disjoint keys or contexts. Sharing a credential context also
shares its budget scope and can enlarge the anonymity set, but activity at
one origin can then reduce the balance available at another.

Anchors constrain new allocations, but the Moderator cannot identify the
issuing Anchor from a redemption. {{MOLE-ARCH}} leaves privacy-preserving
Anchor feedback unresolved. Removing an Anchor requires evidence from outside
that anonymous redemption, and changing sets to probe Clients can reduce
privacy. Endorsements from several accepted Anchors can yield several budgets.

Deployment also needs bounded verification work and replay storage. Invalid
proofs can consume verification capacity, and replay markers must remain while
their Credentials are accepted. Aggregate controls remain necessary for
availability, even though they can reintroduce shared fate.

# Security considerations {#security}

The security goals and limitations of MOLE are described in {{MOLE-ARCH}},
{{MOLE-PROTOCOLS}}, and {{EC-ACT}}.

## Request integrity

An intermediary that sees a presentation can race the Client to spend it.
The request-bound profile of {{MOLE-PROTOCOLS}} must cover every component
that affects authorization, including the body where relevant. Single-use
verification alone determines only which spend wins. Binding does not prevent
an intermediary from withholding or racing an identical request.

A carrier signature should cover the MOLE presentation and the relevant
request components. It attributes that pairing to the carrier. Protection
against the carrier itself requires binding the Client's evidence to the
operation. This document does not define the intermediary signing profile.

Sender-constrained access tokens limit reuse of stolen tokens, but DPoP does
not protect the request body {{DPOP}}. Mutual TLS authenticates the connection
at its TLS endpoint {{MTLS}}, which may be an intermediary. Application-layer
signatures can preserve integrity across TLS termination {{WIMSE-AIMS}}.
For signed HTTP content, the verifier checks Content-Digest against the
received content and verifies a signature covering it, together with the
other components needed to bind the operation {{HTTPSIG}} {{DIGEST}}.

## Correlation and policy

Unlinkable Credentials do not hide addresses, cookies, content, or timing.
A forwarding provider that knows the Client can disclose its requests to the
origin. This deployment does not prevent that collusion.

Credential keys, contexts, and public amounts partition the anonymity set.
They should follow coarse policies shared by many Clients. Per-Client values
or repeated balance probes can make presentations distinguishable. The
privacy requirements of {{MOLE-PROTOCOLS}} apply to allocation, spending,
and refund policy. A platform signature further narrows the observable set.

A Moderator can end a credential chain without identifying its holder. Clients
therefore rely on it to apply the advertised policy. Withholding refunds can
consume scarce Endorsements, while recovery distinguishes a lost response from
a recorded termination only while the result remains available.

# IANA considerations {#iana}

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
