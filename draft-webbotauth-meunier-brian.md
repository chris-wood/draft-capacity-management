---
title: "Bot rate limit is all you need"
abbrev: "Brian"
category: info

docname: draft-webbotauth-meunier-brian-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
# area: "Web and Internet Transport"
# workgroup: "Web Bot Auth"
keyword:
 - not-yet
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
  MOLE-ARCH: I-D.draft-jms-mole-architecture
  WBA-ABA: I-D.draft-ietf-webbotauth-httpsig-protocol
  WBA-HTTPSIG: I-D.draft-rescorla-anonymous-webbotauth

...

--- abstract

This document describes a profile of {{MOLE-ARCH}} that allows for rate limit of clients.
The goal is to allow clients to provide information that sites can use to apply rate limit
policies without requiring a stable identifier.

--- middle

# Introduction {#intro}

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations {#security}

TODO Security


# IANA Considerations {#iana}

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
