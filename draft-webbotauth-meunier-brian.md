---
title: "Bot rate limit is all you need"
abbrev: "Brian"
category: info

docname: draft-webbotauth-meunier-brian-latest
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
  latest: "https://thibmeu.github.io/draft-brian/draft-webbotauth-meunier-brian.html"

author:
 -
    fullname: Thibault Meunier
    organization: Cloudflare
    email: ot-ietf@thibault.uk

normative:

informative:
  AAUTH: I-D.draft-hardt-oauth-aauth-protocol
  CONNECT-UDP: RFC9298
  CONNECT-IP: RFC9484
  OHTTP: RFC9458
  HTTP: RFC9110
  INTERNET-FOR-END-USER: RFC8890
  IPv6: RFC8200
  MOLE-ARCH: I-D.draft-jms-mole-architecture
  MOLE-PROTOCOLS: I-D.draft-jms-mole-protocols
  OAUTH-XAA: I-D.draft-ietf-oauth-identity-assertion-authz-grant
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

...

--- abstract

This document describes a profile of {{MOLE-ARCH}} that allows for rate limiting
of clients. The goal is to allow clients to provide information that sites can
use to apply rate-limit policies without requiring a stable identifier.
Providers endorse their clients, while sites apply anonymous client limits.
{{WBA-HTTPSIG}} can identify intermediaries for separate aggregate limits.

--- middle

# Introduction {#intro}

Agents are increasingly used in business and user workflows, including AI
assistants, search indexing, content aggregation, and automated testing. Their
relationship with websites is symbiotic. They need information from sites, and
sites need signals to apply rate-limit policies. These can be binary, like allow
or block when a site cannot serve traffic, or resource-based to ensure fair access
to everyone.

Historically, most of these signals have been enshrined into deployments of
internet protocols. A client needs an IP address to receive a response from an
origin. Its TLS stack and browser behavior provide further signals. It provides
a familiar User-Agent because middleboxes might otherwise filter its traffic
(see {{USER-AGENT-HISTORY}}).

Recent developments break these patterns. Clients use proxies
({{CONNECT-UDP}}, {{CONNECT-IP}}, {{OHTTP}}, or plain {{HTTP}}).
As intermediaries make requests on behalf of users, the user's TLS stack may no
longer be the one visible to the server. AI agents, local or remote, fetch and
render information in place of browsers. User-Agent strings are still spoofed.

Sites need to apply policy despite these changes. Many approaches rely on
obtaining a stable identifier. An address, including an {{IPv6}} address, is not
necessarily a user. Blocking a shared IP range can affect everyone behind it.
In enterprise settings, a client can identify as a workload {{WIMSE-AIMS}}.
Intermediaries that can tolerate attribution can identify using {{WBA-HTTPSIG}}.
Other approaches involve user authentication or authorization, such as {{AAUTH}}
and {{OAUTH-XAA}}.

While identification solves many problems, it should not be necessary just to
limit a client's consumption. The problem is to give sites that control without
requiring clients to disclose a stable identifier. This supports the user
interests discussed in {{INTERNET-FOR-END-USER}}.

## Motivating use cases

In the cases below, traffic egresses through providers with existing user
relationships. They have means to provide scarce endorsements for their
users, which sites could use to apply rate-limit policies without learning who
the user is.

1. AI agent platforms concentrate traffic on a small set of IP addresses and
   make requests on behalf of users. One user's activity can affect others when
   a site can only limit the shared address or the platform as a whole.
2. Network operators serve users behind shared IP ranges. Policies applied to
   an address range or an entire ASN give these users a shared fate.
3. VPN providers and enterprise ZTNA gateways have a noisy neighbour problem.
   They may know their users, but should not have to expose them to each origin.
   Think of being able to trace requests from a CFO before earnings.

This draft presents an instantiation built on {{MOLE-ARCH}} and how that
composes with {{WBA-HTTPSIG}}. It shares the anonymous rate-limiting goal of
{{WBA-ABA}}. Clients should remain unlinkable through the protocol, even within
the same rate-limit window. Cookies or TLS session tickets can still link
requests independently of MOLE.

## Non use cases

Poppy {{POPPY}} uses User IDs and sessions, including for anonymous browsing.
{{AAP}} uses anonymous admission tokens and recommends a session afterwards,
though it also allows a fresh token per request. Here, the aim is to maintain a
client budget without linking its presentations.

MOLE will not solve every use case served by identifiers. Identity is still
useful, and a story has to be found as to when it is required. Both MOLE and
identity should be provided willingly by the client and by design.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

We use the vocabulary and terminology from {{MOLE-ARCH}}.

# System

There is an intermediate system with MOLE and one Anchor. Essentially, the
provider acts as Anchor and endorses its clients. The origin acts as Moderator.
A Client redeems its Endorsement for a Credential, then presents and updates
that Credential as it makes requests. The origin can apply a client rate limit
or spendable budget. Where an intermediary identifies its forwarded traffic
with Web Bot Auth, the origin can also limit that traffic in aggregate.

One requirement is an Endorsement that supports late binding, such as the
Longfellow approach in {{MOLE-PROTOCOLS}}. The Client can request a Credential
from an accepting Moderator without returning to the Anchor for each
redemption.

~~~aasvg
                        +------------------+
                        |     Provider     |
                        |  +-----------+   |
     +---- Endorsement -+--+   Anchor  |   |
     |                  |  +-----------+   |
     |                  |                  |
     v                  |                  |                  +-----------+
+----------+            |  +-----------+   |                  |   Origin  |
|  Client  +---- MOLE --+->|   Proxy   +---+---- MOLE+WBA --->|           |
+----------+            |  +-----------+   |                  | Moderator |
                        +------------------+                  +-----------+
~~~
{: #single-provider title="Client traffic proxied through its endorsing provider"}

Because of MOLE's issuer-hiding property, this system can evolve to a second
state where multiple providers endorse their clients within the same accepted
Anchor Set. For instance, AI services could form one set, network operators
another, and ZTNA providers a third. The intermediary may still be visible
through Web Bot Auth or its egress address.

~~~aasvg
                        +------------------+
                        |    Provider A    |
                        |  +-----------+   |
     +---- Endorsement -+--+  Anchor A |   |
     |                  |  +-----------+   |
     |                  |                  |
     v                  |                  |
+----------+            |  +-----------+   |
| Client A +---- MOLE --+->|   Proxy   +---+-- MOLE+WBA --+
+----------+            |  +-----------+   |              |
                        +------------------+              |
                                                          |
                                                          |   +-----------+
                                                          |   |   Origin  |
                        +------------------+              +-->|           |
                        |    Provider B    |              |   | Moderator |
                        |  +-----------+   |              |   |           |
                        |  |  Anchor B |   |              |   | Set: A, B |
                        |  +-----+-----+   |              |   +-----------+
                        |        |         |              |
                        |        v         |              |
                        |  +-----------+   |              |
                        |  |  Client B +---+-- MOLE+WBA --+
                        |  +-----------+   |
                        |                  |
                        +------------------+
~~~
{: #multiple-providers title="An external Client and a hosted Client reaching the same origin"}

\[\[ Editor's note: The issuance policy, budget updates, and late-binding details
remain to be worked out. In particular, the same user could obtain Endorsements
from two providers. We rely on their existing client relationships to make
Endorsements scarce, but have not yet specified that assumption. \]\]

# Security Considerations {#security}

TODO Security

# IANA Considerations {#iana}

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
