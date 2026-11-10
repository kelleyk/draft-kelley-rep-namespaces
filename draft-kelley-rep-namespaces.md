---
title: "Namespaces for the Robots Exclusion Protocol"
abbrev: "REP Namespaces"
category: std
docname: draft-kelley-rep-namespaces-latest
updates: 9309
submissiontype: IETF
v: 3
area: "Web and Internet Transport"
keyword:
  - robots.txt
  - crawler
  - product token
  - namespaced label
  - policy token
  - usage preferences
author:
  - fullname: "Kevin Kelley"
    organization: Anthropic
    email: kelleyk-ietf@kelleyk.net

normative:
  RFC2119:
  RFC5234:
  RFC8126:
  RFC8174:
  ROBOTS: RFC9309

informative:
  HTTP: RFC9110
  AIPREF:
    title: "AI Preferences (aipref) Working Group Charter"
    target: https://datatracker.ietf.org/wg/aipref/about/
    author:
      - org: IETF
    date: false
  VOCAB:
    title: "A Vocabulary For Expressing AI Usage Preferences"
    target: https://datatracker.ietf.org/doc/draft-ietf-aipref-vocab/
    seriesinfo:
      Internet-Draft: draft-ietf-aipref-vocab-08
    author:
      - fullname: Paul Keller
      - fullname: Martin Thomson
    date: 2026-09-13
  ATTACH:
    title: "Associating AI Usage Preferences with Content in HTTP"
    target: https://datatracker.ietf.org/doc/draft-ietf-aipref-attach/
    seriesinfo:
      Internet-Draft: draft-ietf-aipref-attach-05
    author:
      - fullname: Gary Illyes
      - fullname: Martin Thomson
    date: 2026-08-18

--- abstract

This document updates RFC 9309, the Robots Exclusion Protocol, so that
a *selector* (`User-Agent:` line) in a robots.txt group can carry a
*label* that is optionally qualified by a *namespace*.  The product
tokens of that standard are described here as unqualified labels;
labels qualified by a namespace are written `__<namespace>_<label>`. It
reserves the sequence `__` for that purpose, establishes a registry of
namespaces, and specifies how namespaced labels are matched. It also
defines terminology for existing parts of robots.txt files. Existing
robots.txt files are interpreted as before, and deployed parsers
ignore namespaced labels.

--- middle

# Introduction

{{ROBOTS}} originated as a mechanism that allowed service operators to
provide machine-readable traffic-management signals to automated
clients; in other words, it allowed websites to direct robots to paths
that they wanted those robots to visit, and away from paths that they
did not.

Because {{ROBOTS}} is the only widely-deployed standard that provides
machine-readable signals to automated clients, it has come to be used
to communicate a variety of other things as well. More such uses are
under active consideration, notably in the IETF AI Preferences
(AIPREF) Working Group {{AIPREF}}. This is understandable, but it also
leads to ambiguity and confusion.

In addition, because {{ROBOTS}} was a de facto standard for decades
before the RFC was written, the parsers actually deployed predate it,
diverge from it, and diverge from one another.  For example, some
merge groups that name the same product token and some do not, some
match product tokens by prefix or substring rather than exactly, some
match line names by prefix, and some end a group at a blank
line.

Changing the format safely is therefore difficult.  Any extension has
to be ignored by every one of those parsers, has to leave unchanged
how every one of them reads the lines that are already there, and has
to mean the same thing to every consumer that does implement it,
without relying on any behavior the deployed parsers disagree about.

In {{ROBOTS}}, a group consists of one or more `User-Agent:` lines,
which this document names *selectors*, followed by one or more rules
(`Allow:` and `Disallow:` lines).  A selector may either be a wildcard
or contain a product token, which is a name that the robot chooses for
itself and that "SHOULD be a substring of the identification string
that the crawler sends to the service" ({{Section 2.2.1 of ROBOTS}}).

This document updates {{ROBOTS}} to generalize non-wildcard selectors
so that each carries a *label* that is optionally qualified by a
*namespace*.  An unqualified label is in the default namespace; it is
exactly the product token of {{ROBOTS}} and keeps its existing
meaning. A *namespaced label* is written `__<namespace>_<label>`.

Each namespace is registered with IANA, and the specification that
registered it defines what labels the namespace contains and their
semantics ({{labels}}).

A namespaced label stays within the alphabet that {{ROBOTS}} allows
for product tokens (`a-z`, `A-Z`, `_`, `-`). This document therefore
introduces no new directive names and no new characters; the path
language, longest-match precedence, and group structure defined by
{{ROBOTS}} are not changed.

As a result, the wide variety of existing parsers deployed across the
Internet will continue to interpret traffic-management signals as they
did previously, and will ignore selectors for a namespaced label as
they would any selector for a product token they do not recognize.  In
addition, a group's selectors can freely combine namespaced labels
from one or more namespaces with product tokens.  This avoids
unnecessary duplication in robots.txt files, which are
size-constrained.

Equally importantly, people who author robots.txt files, and people
who build software that consumes them, keep the semantics they already
know: a group for a namespaced label is written, read, and evaluated
like any other group, so the mental model they already have carries
over unchanged.

## Motivation {#motivation}

The immediate need for namespaced labels comes from the AIPREF
Working Group, which is defining a vocabulary of
usage preferences {{VOCAB}} and means of associating those preferences
with content {{ATTACH}}, including in robots.txt.

The goal of that work is to allow a service operator to bind usage
preferences (e.g., "no AI training") to paths.

Such a preference is about use, not a traffic-management signal; the
two are intentionally separate. Ordinary rules for a crawler's own
product token still govern fetching, and nothing about the preference
changes that signaling; the two types of signals are independent and
orthogonal.

The Working Group's attachment needs could be addressed by registering
a `usage` namespace, with labels corresponding to the vocabulary terms
it standardizes.

A label used this way is an example of a *policy token*
{{policy-tokens}}.

## Example {#example}

This section is not normative. It uses the `example` namespace, which
is reserved for documentation ({{iana}}), and treats it as though a
specification had defined labels `foo` and `bar` in it.

~~~
User-Agent: *
Disallow: /private/

User-Agent: ExampleBot
Disallow: /search/

User-Agent: __example_foo
Disallow: /articles/
~~~

The first two groups are ordinary: the crawler `ExampleBot` obeys the
second, and other crawlers the first. The third group's selector
carries the namespaced label `__example_foo`. A consumer that
implements the namespace looks that label up exactly as a crawler
looks up its product token, and finds `/articles/` disallowed and
every other path allowed; what "allowed" and "disallowed" express is
defined by the namespace's specification ({{defining}}). Had the file
no third group, that specification would also decide whether the first
group is consulted instead ({{matching}}). A parser that does not
implement this document sees a group for a product token that no
crawler uses, and ignores it.

~~~
User-Agent: ExampleBot
User-Agent: __example_bar
Disallow: /drafts/

User-Agent: Example__Bot
Disallow: /
~~~

Here the first group has two selectors and matches both the product
token `ExampleBot` and the namespaced label `__example_bar`
({{mixed}}). The second group's selector is invalid, because its value
contains `__` and is not a well-formed namespaced label. It matches no
label, so `Disallow: /` applies to nothing, but it still begins a new
group: that rule is not added to the group before it ({{reserved}}).

# Conventions and Definitions {#conventions}

{::boilerplate bcp14-tagged}

This document uses the ABNF of {{RFC5234}}, including its core rule
ALPHA, and the rule `identifier` from {{Section 2.2 of ROBOTS}}.

service operator:
: An entity that operates a service, such as a website, and publishes
  a robots.txt file for it. {{ROBOTS}} calls this party a "service
  owner".

consumer:
: An entity that interprets signals that a service operator expresses
  in robots.txt by looking up a product token or namespaced label.
  Consumers are often crawlers or other types of robots.

parser:
: In the sense of {{ROBOTS}}; software that parses robots.txt files
  and matches labels (e.g. product tokens) and paths against their
  rules.

identification string:
: In the sense of {{Section 2.2.1 of ROBOTS}}; the string a crawler
  sends to identify itself. In the case of HTTP, the `User-Agent`
  header field ({{Section 10.1.5 of HTTP}}).

selector:
: A `User-Agent:` line ({{Section 2.2.1 of ROBOTS}}; `startgroupline`
  in the grammar of {{Section 2.2 of ROBOTS}}). A selector carries a
  label or the value `*`.

wildcard selector:
: A selector that carries the value `*` (i.e., `User-Agent: *`).
  {{ROBOTS}} calls this 'a user-agent line with the "*" value'.

wildcard fallback:
: The behavior of {{Section 2.2.1 of ROBOTS}} under which a consumer
  that finds no group matching a label obeys the group(s) with a
  wildcard selector instead. The fallback applies only when no other
  group matches the label; the rules of a wildcard selector's group are
  never combined with those of a group that does match it. A group
  whose selectors include both `*` and the label in question matches
  that label through the latter, and is not a use of the fallback.

robot:
: An automated client operated by a single party and whose behavior is
  controlled, directly or indirectly, by that party.  Crawlers are
  classic examples of robots; browsers and other agents acting on
  behalf of a user are not.

rule:
: As in {{Section 2.1 of ROBOTS}}; an `Allow:` or `Disallow:` line.

group:
: One or more selectors followed by one or more rules ({{Section 2.1
  of ROBOTS}}). The selectors within a group are disjunctive; in other
  words, a group *matches* a label if any of its selectors does
  ({{mixed}}).

label:
: A name within a namespace; what a selector carries, other than `*`
  ({{syntax}}). Every label belongs to exactly one namespace: an
  unqualified label, which is a product token, belongs to the default
  namespace, and a qualified label to the namespace that qualifies it.
  The qualifier is not part of the label. Labels MUST NOT contain the
  sequence `__`.

namespace:
: A set of labels defined by one specification. The *default
  namespace* is the set of unqualified labels; it is defined by
  {{Section 2.2.1 of ROBOTS}}, has no name, and is not registered.
  Every other namespace has a registered name that qualifies its
  labels, and the specification that registers it defines the labels
  in it and their meaning ({{defining}}, {{iana}}).

product token:
: In the sense of {{Section 2.2.1 of ROBOTS}}; the name that a crawler
  chooses for itself and uses to find the groups that apply to it. In
  the terms of this document, a product token is an unqualified label,
  which is to say a label in the default namespace: "product token",
  "unqualified label", and "label in the default namespace" all name
  the same thing. This document does not change what a product token
  means or the alphabet it is drawn from (letters, `_`, `-`); it gives
  the `product-token` grammar rule that same meaning ({{syntax}}).

namespaced label:
: A label in a namespace other than the default namespace, written
  together with the namespace that qualifies it as
  `__<namespace>_<label>`; the `namespaced-label` rule of {{syntax}}.
  The term names the whole qualified value; the label itself is the
  part that follows the namespace (`ns-label`).

invalid selector:
: A selector whose value is not a `selector-value` ({{syntax}}).

policy token:
: A label, whether a product token or a namespaced label, that a
  consumer looks up regardless of whether it appears in the
  identification string of any outbound request. Existing examples
  describe how content will be used rather than whether it may be
  fetched ({{policy-tokens}}).

proprietary policy token:
: A policy token that is a product token documented by a single
  consumer. Some service operators already include these in their
  robots.txt files. The consumer that defines a proprietary policy
  token SHOULD clearly and publicly describe how it interacts with
  other mechanisms that consumer supports ({{policy-tokens}}).

# Labels and Namespaces {#labels}

This section updates {{Sections 2.1, 2.2, and 2.2.1 of ROBOTS}}.

A group is one or more selectors followed by one or more rules. A
selector carries either the value `*` or a label, and every label
belongs to a namespace. An unqualified label belongs to the *default
namespace* and is a product token exactly as {{ROBOTS}} describes; the
default namespace is defined by {{ROBOTS}} and nothing in this
document changes it. A qualified label belongs to the registered
namespace that qualifies it.

## Syntax {#syntax}

The grammar of {{Section 2.2 of ROBOTS}} uses the rule name
`product-token` for the whole value of a `User-Agent:` line, including
the value `*`, although its prose uses "product token" only for the
name of a crawler. This document brings the grammar into line with the
prose: the value of a selector is a wildcard, a product token, or a
namespaced label, and `product-token` names only the second of those.

The `startgroupline` and `product-token` rules of {{Section 2.2 of
ROBOTS}} are replaced by the following rules; its `identifier` rule is
unchanged.

~~~ abnf
startgroupline   = *WS "user-agent" *WS ":" *WS
                   ( selector-value / invalid-value ) EOL

selector-value   = "*" / product-token / namespaced-label
invalid-value    = identifier  ; that is not a selector-value

product-token    = word *( "_" word )
word             = 1*( ALPHA / "-" )

namespaced-label = "__" namespace "_" ns-label
namespace        = ALPHA [ *( ALPHA / "-" ) ALPHA ]
ns-label         = ns-word *( "_" ns-word )
ns-word          = ALPHA [ *( ALPHA / "-" ) ALPHA ]
~~~

A selector whose value is a `selector-value` is valid: `*` makes it a
wildcard selector, a `product-token` is a product token in the sense
of {{ROBOTS}}, and a `namespaced-label` is a namespaced label in the
named namespace, whose `ns-label` part is the label. A selector
whose value is an `invalid-value` is an *invalid selector*; it begins
or continues a group exactly as any other `startgroupline` does, and
matches no label ({{reserved}}).

The set of lines that `startgroupline` accepts is unchanged: every
`selector-value` is an `identifier` or `*`. The invalid values are the
`identifier`s that contain `__` other than as a well-formed
`namespaced-label`, or that begin or end with `_`. The grammar of
{{ROBOTS}} admitted these as product tokens and this document does
not; no product token in use is known to be among them
({{deployment}}).

No namespace and no label contains `__`, and a namespace contains no
`_` at all, so the leading `__` unambiguously introduces a
namespace and the first `_` after it unambiguously ends it. Namespaces
and labels, product tokens included, are compared case-insensitively.

For reference, the complete formal syntax of {{Section 2.2 of ROBOTS}}
as amended by this document is reproduced below.

~~~ abnf
robotstxt = *(group / emptyline)
group = startgroupline                ; We start with a user-agent
                                      ; line
       *(startgroupline / emptyline)  ; ... and possibly more
                                      ; user-agent lines
       *(rule / emptyline)            ; followed by rules relevant
                                      ; for the preceding
                                      ; user-agent lines

; Changed by this document: the value was product-token, which
; was defined as identifier / "*".
startgroupline = *WS "user-agent" *WS ":" *WS
                 ( selector-value / invalid-value ) EOL

rule = *WS ("allow" / "disallow") *WS ":"
      *WS (path-pattern / empty-pattern) EOL

; parser implementors: define additional lines you need (for
; example, Sitemaps).

path-pattern = "/" *UTF8-char-noctl   ; valid URI path pattern
empty-pattern = *WS

identifier = 1*(%x2D / %x41-5A / %x5F / %x61-7A)

; Changed or added by this document. A startgroupline that
; carries an invalid-value is an invalid selector; it still
; delimits a group, and matches no label.
selector-value   = "*" / product-token / namespaced-label
invalid-value    = identifier  ; that is not a selector-value
product-token    = word *( "_" word )
word             = 1*( ALPHA / "-" )
namespaced-label = "__" namespace "_" ns-label
namespace        = ALPHA [ *( ALPHA / "-" ) ALPHA ]
ns-label         = ns-word *( "_" ns-word )
ns-word          = ALPHA [ *( ALPHA / "-" ) ALPHA ]

comment = "#" *(UTF8-char-noctl / WS / "#")
emptyline = EOL
EOL = *WS [comment] NL ; end-of-line may have
                       ; optional trailing comment
NL = %x0D / %x0A / %x0D.0A
WS = %x20 / %x09

; UTF8 derived from RFC 3629, but excluding control characters

UTF8-char-noctl = UTF8-1-noctl / UTF8-2 / UTF8-3 / UTF8-4
UTF8-1-noctl = %x21 / %x22 / %x24-7F ; excluding control, space, "#"
UTF8-2 = %xC2-DF UTF8-tail
UTF8-3 = %xE0 %xA0-BF UTF8-tail / %xE1-EC 2UTF8-tail /
         %xED %x80-9F UTF8-tail / %xEE-EF 2UTF8-tail
UTF8-4 = %xF0 %x90-BF 2UTF8-tail / %xF1-F3 3UTF8-tail /
         %xF4 %x80-8F 2UTF8-tail

UTF8-tail = %x80-BF
~~~
{: title="Formal syntax of RFC 9309 as amended"}

## Groups with Multiple Selectors {#mixed}

A single group MAY have several selectors ({{Section 2.2 of ROBOTS}}),
and those selectors MAY carry labels from the same or different
namespaces, including the default namespace (i.e., they may be product
tokens rather than namespaced labels).

As in {{ROBOTS}}, selectors in a group are disjunctive; in other
words, the group matches a label if any one of its selectors does.
This is a consequence of the need to be backwards-compatible with
deployed software: a parser that does not implement this document
applies a group to the product tokens it names exactly as {{ROBOTS}}
specifies, whatever else the group's selectors carry. Accordingly, no
specification defining a namespace can alter whether a group matches a
product token or what its rules mean for one.

A specification defining a namespace MAY describe other behavior among
its own labels, or between its labels and those of another namespace,
when they appear together in a group or across groups. A specification
that does so MUST specifically account for consumers that do not
implement this document, for consumers that implement only some of the
namespaces involved, and for the disjunctive reading that such
consumers will apply to whatever they do recognize, and MUST show that
the file means something acceptable to its author under each of those
readings. It SHOULD NOT make the meaning of its labels depend on which
product tokens share their group, since that would scope the namespace
by crawler through group membership and invite service operators to
add product tokens to a group, creating crawl rules they did not
intend.

## Reserved Selector Values {#reserved}

Every `identifier` that contains the sequence `__` at any position, or
that begins or ends with `_`, is reserved to this document and to the
namespaces registered under it. Such a value is not a product token;
it either parses as a `namespaced-label` or makes its selector an
invalid selector ({{syntax}}).

The reservation covers `__` at any position, not only as a prefix,
because some deployed parsers match more flexibly than {{ROBOTS}}
specifies.  Specifically, they treat a selector as matching when the
value it carries is a prefix of, or a substring of, the crawler's
product token or its whole identification string. Were only the prefix
reserved, a crawler could name itself `Foo__example_foo`, and under
substring matching the group for the namespaced label `__example_foo`
would be applied to that crawler.  Reserving the sequence everywhere
makes accidental matches implausible without requiring any deployed
parser to change.

The reservation cannot help where a parser matches in the other
direction, treating a selector as matching when the crawler's product
token is a prefix or a substring of the value the selector carries.
Under such a parser, a crawler named `example` or `foo` would match a
selector carrying `__example_foo`.  Because of substring matching, a
namespace name SHOULD be compared against a corpus of known or likely
product tokens before it is registered ({{expert-guidance}}), and so
SHOULD each label before it is defined in a namespace ({{defining}}).

No label contains `__`, whichever namespace it is in. In particular, a
product token MUST NOT contain `__`, the label in a namespaced label
(its `ns-label` part) MUST NOT contain `__`, and a namespace MUST NOT
contain `_` at all. Using `-` as the word separator in both namespaces
and labels is RECOMMENDED.

A crawler SHOULD NOT send an identification string that contains `__`.

A consumer MUST NOT document a proprietary policy token that contains
`__` ({{policy-tokens}}).

A consumer MUST treat an invalid selector as matching no label, and
MUST otherwise process it as it would any other selector; it still
begins or continues a group, and the rules that follow it belong to
that group.

A consumer MUST likewise treat a selector carrying a well-formed
namespaced label as matching no label when it does not recognize the
namespace or does not implement it.

Service operators MUST NOT publish robots.txt files that contain
invalid selectors.

# Defining a Namespace {#defining}

The default namespace is defined by {{ROBOTS}} and is outside the
scope of this section: its labels are product tokens, the wildcard
fallback applies to them, and allowed and disallowed mean what
{{Section 2.2.2 of ROBOTS}} says. Every other namespace identifies a
specification, recorded in the registry of {{iana}}, and every label
in it is defined by, and only by, that specification. Beyond the
syntax of {{syntax}} and the exact matching of {{matching}}, which
hold for every namespace, that specification decides the remaining
questions, including whether the wildcard fallback applies to its
labels. This section lists them.

Label syntax:
: The specification MUST define the syntax of labels within its
  namespace, within the `ns-label` rule of {{syntax}}, and MAY
  establish a registry of labels. Because of substring matching, each
  label SHOULD be compared against a corpus of known or likely product
  tokens before it is defined or registered, and SHOULD neither be one
  nor contain one ({{reserved}}).

Wildcard fallback:
: The specification MUST state whether the wildcard fallback applies to
  labels in its namespace ({{matching}}): always ("yes"), never ("no"),
  or under conditions the specification sets out
  ("specification-defined"). Where it applies, it is the fallback of
  {{Section 2.2.1 of ROBOTS}} and nothing more: a consumer that finds
  no group matching the namespaced label obeys the group(s) with a
  wildcard selector, exactly as a crawler whose product token matches
  no group does, and a consumer that does find a matching group does
  not consult wildcard groups at all. Where it does not apply, a
  consumer that finds no matching group has been given no signal (see
  "No matching selector" below). Nearly every robots.txt file published
  today has a group with a wildcard selector, written with crawling in
  mind and without knowledge of any namespace. A namespace whose labels
  express something the authors of those groups cannot have intended
  SHOULD specify that the wildcard fallback does not apply to its
  labels, so that a file written without knowledge of the namespace
  says nothing in it; the `usage` namespace anticipated in
  {{motivation}} is expected to take this position. A namespace MAY
  instead specify that the wildcard fallback does apply, where reading
  existing wildcard groups under it is the intended effect; its
  specification SHOULD then explain why that reading is safe for files
  written before it existed. A namespace whose answer depends on the
  label, on the content of the file, or on anything else MUST set out
  those conditions completely, such that two consumers reading the
  same file reach the same answer.

Allowed and disallowed:
: The specification MUST define what is expressed when lookup finds a
  path allowed, and when it finds a path disallowed, in the group(s)
  that match a namespaced label in its namespace ({{matching}}).

No matching selector:
: The specification MAY define what is expressed when no selector
  matches such a namespaced label ({{matching}}).  If it does not,
  then the robots.txt file expresses nothing in the context of that
  specification.

No matching rule:
: The specification MAY define what, if anything, is expressed when a
  selector matches such a namespaced label but no rule in the matching
  group(s) matches the path.  If it does not, then per {{ROBOTS}}, the
  expression is equivalent to an `Allow:` rule.

Interaction with other labels:
: The specification MAY describe behavior other than disjunction among
  its own labels or between its labels and those of other namespaces,
  within the limits of {{mixed}}: it MUST account for consumers that
  do not implement this document or that implement only some of the
  namespaces involved, it MUST NOT alter whether a group matches a
  product token or what a group's rules mean for one, and it SHOULD
  NOT make the meaning of its labels depend on which product tokens
  share their group.

Relationship to rules for product tokens:
: The specification SHOULD state how rules under its labels relate to
  rules under a consumer's own product token(s). For a namespace used
  for policy tokens, {{policy-tokens}} supplies that statement.

This document registers no namespace for use. It reserves the
namespace `example` for documentation ({{iana}}).

# Matching Namespaced Labels {#matching}

This section updates {{Section 2.2.1 of ROBOTS}}.

To determine what has been expressed for a given path under a
namespaced label, a consumer looks up the path and label combination
as it would the combination of a path and a product token under
{{Sections 2.2.1 and 2.2.2 of ROBOTS}}, subject to the following.

Consumers MUST match namespaced labels exactly, apart from case,
regardless of their matching behavior for product tokens.  Prefix
matches and substring matches MUST NOT be applied.

If there is more than one group matching the label, the matching
groups' rules are combined as {{Section 2.2.1 of ROBOTS}} specifies.

If no group in the file matches the namespaced label, the namespace's
specification decides whether the wildcard fallback applies
({{defining}}). Where it applies and the file has a wildcard selector,
the consumer obeys the group(s) with a wildcard selector as {{Section
2.2.1 of ROBOTS}} specifies for a product token that matches no group,
and evaluates the path against their rules as described below. Where
it does not apply, or the file has no wildcard selector, no signal has
been provided, and the namespace's specification gives that outcome
its meaning ({{defining}}).

A consumer MUST NOT consult a wildcard selector's group for a
namespaced label in any other circumstance. In particular, it MUST NOT
apply the wildcard fallback where the namespace's specification does
not provide for it, and it MUST NOT combine the rules of a wildcard
selector's group with those of a group that matches the label.

If the lookup finds that the path is disallowed, or finds that it is
allowed, what has been expressed is likewise defined by the
namespace's specification. Path evaluation itself is exactly that of
{{Section 2.2.2 of ROBOTS}}, including longest-match precedence, the
special characters of {{Section 2.2.3 of ROBOTS}}, and the use of the
`allow` rule when an `allow` and a `disallow` rule are equivalent.

Implementers are cautioned that general-purpose robots.txt parsers
apply the wildcard fallback to any label they find no other match for,
and that several match by prefix or substring. A consumer built on
such a parser MUST configure or wrap it to comply with this section.

# Policy Tokens {#policy-tokens}

This section updates {{Section 2.2.1 of ROBOTS}}.

A policy token is a label, whether a product token or a namespaced
label, whose RFC 9309 signals a consumer looks up regardless of
whether it appears in the identification string of any outbound
request ({{conventions}}). The recommendation in {{Section 2.2.1 of
ROBOTS}} that a product token be a substring of the identification
string applies to a product token that names a crawler; it does not
apply to a policy token.

Some crawler operators already publish policy tokens, e.g. ones that
control how content is used rather than which crawler can fetch it.
These vendor-specific policy tokens are called *proprietary policy
tokens*.

Consumers SHOULD interpret a proprietary policy token as they would in
the absence of this document. A consumer that documents a proprietary
policy token SHOULD document whether the wildcard fallback applies to
it, how its rules relate to rules for that consumer's crawler product
tokens, and how they relate to any namespaces the consumer implements.

## Independence from Crawler Rules {#independence}

Consumers MUST interpret the RFC 9309 signals being expressed based on
their own product token(s) exactly as they would in the absence of
this document. That is, `ExampleBot` still checks whether there are
rules for `ExampleBot`; if there are, they still have the meaning
described in {{ROBOTS}}.

Signals expressed through product tokens and through namespaced labels
may coexist in the same robots.txt file, but they are independent and
orthogonal unless the specification governing a namespace says
otherwise ({{defining}}).

# Deployment Considerations {#deployment}

A parser that implements only {{ROBOTS}} reads a namespaced label as
it would any product token, finds that it matches no crawler it is
asked about, and ignores the group as it would any group for a
product token it does not recognize; where the group also has selectors
carrying product tokens, it applies the group to those exactly as
written. No deployed file changes meaning, because no deployed file is
known to use a reserved value: a survey of widely fetched robots.txt
files conducted for this document found no selector value that
contains `__` or begins or ends with `_`.

The constraints in {{reserved}} on crawler product tokens and
identification strings cost existing crawlers nothing for the same
reason: none is known to use `__` today. They exist to keep it that
way.

Service operators, and tools that check robots.txt files for them,
SHOULD flag a namespaced label whose namespace is not registered, and
SHOULD flag an invalid selector ({{syntax}}), as these match nothing
for any conforming consumer.

# Security Considerations {#security}

The security considerations of {{Section 3 of ROBOTS}} apply to this
mechanism.

# Privacy Considerations {#privacy}

This document does not introduce any new privacy considerations.

# IANA Considerations {#iana}

## REP Namespaces Registry

IANA is requested to create a new registry titled "REP Namespaces" in
a new "Robots Exclusion Protocol (REP)" registry group.

The registration policy is Specification Required ({{Section 4.6 of
RFC8126}}).

Each registration consists of:

Namespace:
: The `namespace` string, conforming to {{syntax}}. Namespaces are
  compared case-insensitively and recorded in lowercase.

Label syntax:
: The syntax of labels within this namespace, within the `ns-label`
  rule of {{syntax}}, or a reference to the registry that enumerates
  permitted labels.

Wildcard fallback:
: One of "yes", "no", or "specification-defined": whether the wildcard
  fallback of {{Section 2.2.1 of ROBOTS}} applies to labels in this
  namespace when no other group matches them ({{defining}}). For
  "specification-defined", the referenced specification MUST be
  consulted.

Reference:
: The specification that defines the namespace as {{defining}}
  requires.

Change controller:
: The party to whom IANA refers requests to change the registration.
  For registrations made by IETF Stream documents, the IETF
  (iesg@ietf.org); otherwise, the person or organization that made the
  registration, or a successor it designates.

The default namespace ({{conventions}}) has no name and no entry in
this registry.

The registry initially contains one entry, which reserves the
namespace `example` for use in documentation and examples:

Namespace:
: example

Label syntax:
: Reserved; any `ns-label`.

Wildcard fallback:
: Reserved; not applicable.

Reference:
: This document.

Change controller:
: IETF (iesg@ietf.org)

The `example` namespace has no defined labels and expresses nothing. A
consumer treats a selector that carries one of its labels as it does a
selector for any namespace it does not implement ({{reserved}}).

The first registration of a namespace for use is expected to be
`usage`, made by the AI Preferences Working Group's attachment work
({{motivation}}).

### Guidance for Designated Experts {#expert-guidance}

{{Section 4.6 of RFC8126}} asks that a registry using Specification
Required give its designated experts clear review criteria. The
expert(s) review a request against the following:

* The referenced specification is permanent and readily available,
  and defines the namespace as {{defining}} requires, addressing each
  item listed there rather than leaving any to implementers'
  discretion.
* The namespace name conforms to the `namespace` rule of {{syntax}},
  is recorded in lowercase, and is neither identical to nor
  confusingly similar to a registered namespace.
* The namespace name has been compared against a corpus of known or
  likely product tokens, and neither is one nor contains one
  ({{reserved}}).
* The label syntax stays within the `ns-label` rule of {{syntax}}.
* The "Wildcard fallback" field agrees with the specification, and,
  for "specification-defined", the specification sets out the
  conditions completely, such that two consumers reading the same
  file reach the same answer ({{defining}}).
* Where the specification describes interaction with other labels, it
  meets the requirements of {{mixed}}.

The expert(s) do not assess the merits of what a namespace's labels
express; that is for the review process of the specification itself.
They MAY ask for a different name where the requested one is likely
to mislead, and SHOULD decline a request whose specification does not
meet the criteria above, explaining which it fails.

--- back

# Acknowledgments
{:numbered="false"}

This work was informed by conversations with many members of the AI
Preferences (AIPREF) Working Group and other members of the IETF
community.  Thank you all, and please reach out to the author so that
your name can be listed here.
