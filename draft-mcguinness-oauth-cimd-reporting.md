---
title: "OAuth Client ID Metadata Document Reporting"
abbrev: "CIMD Reporting"
category: std

docname: draft-mcguinness-oauth-cimd-reporting-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - OAuth
 - client metadata
 - Client ID Metadata Document
 - reporting
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "mcguinness/draft-mcguinness-oauth-cimd-reporting"
  latest: "https://mcguinness.github.io/draft-mcguinness-oauth-cimd-reporting/draft-mcguinness-oauth-cimd-reporting.html"

author:
 -
    fullname: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
  RFC3339:
  RFC3986:
  RFC6454:
  RFC6585:
  RFC6749:
  RFC6838:
  RFC7515:
  RFC7591:
  RFC8126:
  RFC8259:
  RFC8414:
  RFC9110:
  RFC9530:
  RFC9562:
  CIMD: I-D.ietf-oauth-client-id-metadata-document
  IANA.oauth-parameters:
    title: "OAuth Parameters"
    target: https://www.iana.org/assignments/oauth-parameters
    author:
      org: IANA
  IANA.media-types:
    title: "Media Types"
    target: https://www.iana.org/assignments/media-types
    author:
      org: IANA

informative:
  RFC8460:
  RFC9111:
  RFC9421:
  RFC9990:

--- abstract

This specification defines how an OAuth authorization server reports
problems with a client's Client ID Metadata Document to the party that
publishes it, and how it evaluates a candidate replacement document
before the publisher serves that document at the client's Client
Identifier URL. Each report describes the results of named checks and,
when a document was obtained, identifies the exact document evaluated.
Reporting is asynchronous and best effort. It does not change how the
authorization server processes OAuth requests, and neither a report nor
its absence establishes that a client is authorized or that a metadata
change is safe to deploy.

--- to_be_removed_note_Open_Issues

The following questions are open. Each is tracked as an issue in the
[GitHub repository](https://github.com/mcguinness/draft-mcguinness-oauth-cimd-reporting/issues),
where discussion is welcome.

* Reporting before first acceptance (#1): should a document that the
  reporter does not accept be able to name a collector for reports about
  its own rejection?

* Cross-origin collectors (#2): what mechanism would let a collector on
  another origin, or on a shared origin, authorize the reports it
  receives?

* Reporter authentication (#3): should reports be signed, given that some
  reporters prefer not to identify themselves?

* Advertising support (#4): should authorization servers advertise
  reporting support in their metadata?

* Operational parameters (#5): are the report size limit, delivery window,
  reporting period, candidate report interval, and configuration
  retention period right?

* Granularity of results (#6): how much should reason codes disclose about
  a reporter's policy?

* Request-level reports (#7): should a later version report request-time
  failures attributable to metadata?

* Referenced resources (#8): should reports carry digests of resources,
  such as a JWK Set, that an evaluation used?

* Conditional requests (#9): how should a 304 response be treated for
  active documents, given that Section 6 of {{CIMD}} treats every status
  code other than 200 as an error?

* Applicability (#10): should reporting extend to clients registered
  through Dynamic Client Registration or to software statements?

* Candidate evaluation (#11): does it justify its complexity, should it be
  specified separately, and is one candidate URL retrieved by the
  reporter the right model?

--- middle

# Introduction {#intro}

An OAuth Client ID Metadata Document {{CIMD}} allows a client to use an
HTTPS URL as its client identifier. An authorization server retrieves
the client's metadata from that URL ({{Section 6 of CIMD}}) and decides,
according to its own supported values and policy, whether it can use
that metadata. One document therefore serves every authorization server
the client interacts with, and each of those servers can reach a
different conclusion about it.

When an authorization server cannot use the document, because
retrieval fails, the document is invalid, or a value is unsupported or
not permitted by local policy, the failure is visible to the end user
and to the operator of that authorization server. The publisher of the
document is not informed. A change to the document can therefore cause
the client to fail at authorization servers that the publisher cannot
observe, and the publisher cannot test a proposed change against those
servers before deploying it.

This specification defines two optional functions that an
authorization server can perform for a client that requests them:

1. Reporting problems with the active document, which is the document
   served at the Client Identifier URL.

2. Evaluating a candidate document, which is a proposed replacement
   served at a separate URL, and reporting the results without applying
   the candidate to any OAuth request.

Both functions retain one active document for each Client Identifier
URL; serving a document at the Client Identifier URL remains the only
way to change the client's metadata. Neither function changes how the
authorization server processes OAuth requests ({{independence}}). The
approach follows SMTP TLS Reporting {{RFC8460}}, in which sending
systems report failures to recipient domains that cannot otherwise
observe them, as a companion to the specifications that define the
policies themselves.

## Scope {#scope}

This specification applies to clients identified by a Client
Identifier URL {{CIMD}}. Establishing reporting configuration for
clients registered by other means, such as Dynamic Client Registration
{{RFC7591}}, is out of scope. Reports describe Client ID Metadata
Documents, not individual OAuth requests.

# Conventions and Definitions {#conventions}

{::boilerplate bcp14-tagged}

This specification uses the terms "authorization server" and "client"
defined in {{RFC6749}}, and the terms "Client Identifier URL" and
"Client ID Metadata Document" defined in {{CIMD}}.

This specification also defines the following terms:

publisher:
: The party that controls the content served at a Client Identifier
  URL.

reporter:
: An authorization server that evaluates Client ID Metadata Documents
  and delivers reports as described in this specification. Each
  authorization server with its own issuer identifier ({{RFC8414}}) is a
  separate reporter, even when several share an implementation or
  infrastructure.

collector:
: The HTTPS endpoint to which a reporter delivers reports for a client.
  Its URL, the collector URL, is the value of the client's
  `client_metadata_report_uri` member.

active document:
: The document served at a Client Identifier URL, as obtained by a
  reporter that retrieves that URL as described in
  {{Section 6 of CIMD}}. Being active does not imply that the reporter
  accepts the document.

accept:
: A reporter accepts an active document when it uses that document as
  the client's metadata to process OAuth requests, as described in
  {{CIMD}}.

candidate document:
: A document that a reporter retrieves, solely for evaluation, from the
  candidate URL, which is the value of the client's
  `client_metadata_candidate_uri` member.

check:
: A named test applied to a document, or to an attempt to retrieve a
  document or a resource it references, that produces an outcome
  ({{checks}}).

evaluation:
: The application of one or more checks to one retrieval of a document.

result:
: The outcome of one check, optionally scoped to one member of the
  document, as recorded by a reporter and conveyed in a report
  ({{format}}).

reporting period:
: The interval, chosen by the reporter, in which it observed the
  results that a report contains.

reporting configuration:
: The collector URL, and the candidate URL if any, that a reporter has
  established for a Client Identifier URL ({{establishing}}).

# Protocol Overview {#overview}

{{fig-overview}} shows the interactions this specification adds to the
retrieval of a Client ID Metadata Document. The active document, the
candidate document, and the collector are all on the origin of the
Client Identifier URL.

~~~ ascii-art
  Publisher                  Reporter                   Collector
      |                          |                          |
      |   (A) GET Client Identifier URL                     |
      |<-------------------------|                          |
      |   active document        |                          |
      |------------------------->|                          |
      |                          | accept; establish        |
      |                          | reporting configuration  |
      |                          |                          |
      |   (B) GET candidate URL (asynchronous)              |
      |<-------------------------|                          |
      |   candidate document     |                          |
      |------------------------->|                          |
      |                          | evaluate                 |
      |                          |                          |
      |                          |   (C) POST report        |
      |                          |------------------------->|
      |                          |   2xx                    |
      |                          |<-------------------------|
~~~
{: #fig-overview title="Reporting and Candidate Evaluation"}

(A) The reporter retrieves and processes the active document as
    specified in {{CIMD}}. If the reporter accepts the document and the
    document requests reporting, the reporter establishes reporting
    configuration ({{establishing}}).

(B) If the reporting configuration includes a candidate URL, the
    reporter can retrieve and evaluate the candidate document outside
    the processing of any OAuth request ({{candidate}}).

(C) The reporter delivers reports about the active document, the
    candidate document, or both, to the collector ({{delivery}}).

A publisher can use candidate evaluation to stage a change: serve the
proposed document at the candidate URL and name that URL in the active
document; review candidate reports that carry the proposed document's
digest; serve the proposed document at the Client Identifier URL; and
then review active document reports that carry the new digest.

## Independence from Request Processing {#independence}

A reporter MUST determine the outcome of an OAuth request independently
of whether reporting or candidate evaluation is enabled, succeeds, or
fails. The members defined in {{metadata}}, any candidate document, the
collector, and any report have no effect on whether the reporter
accepts the active document, on client authentication, or on any other
authorization decision.

Candidate retrieval, candidate evaluation, and report delivery MUST NOT
delay the response to an OAuth request.

# Client Metadata {#metadata}

This specification defines the following client metadata members for
use in a Client ID Metadata Document:

`client_metadata_report_uri`:
: OPTIONAL. URL of the collector to which reporters deliver reports
  about this client's metadata.

`client_metadata_candidate_uri`:
: OPTIONAL. URL of a candidate document that the publisher asks
  reporters to evaluate. This URL MUST NOT be identical, under simple
  string comparison, to the Client Identifier URL. A reporter MUST
  ignore this member if `client_metadata_report_uri` is absent.

The value of each member MUST be an absolute URL that uses the `https`
scheme, has the same origin ({{Section 5 of RFC6454}}) as the Client
Identifier URL, and has no fragment component. Origin comparison differs
from the simple string comparison that {{Section 3 of CIMD}} requires
for Client Identifier URLs: for example,
`https://client.example:443/reports` has the same origin as the Client
Identifier URL `https://client.example/oauth/client.json`. A reporter
MUST treat a member whose value does not meet the requirements of this
section as absent.

{{Section 10.7 of CIMD}} applies to requests that a reporter makes to
these URLs as it applies to any URL contained in a Client ID Metadata
Document. An authorization server that does not implement this
specification is expected to ignore both members, consistent with the
treatment of client metadata that an authorization server does not
understand in {{Section 2 of RFC7591}}.

The following is an example of a Client ID Metadata Document that
requests reporting and candidate evaluation:

~~~ json
{
  "client_id": "https://client.example/oauth/client.json",
  "client_name": "Example Client",
  "redirect_uris": ["https://client.example/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "none",
  "client_metadata_report_uri":
    "https://client.example/oauth/reports",
  "client_metadata_candidate_uri":
    "https://client.example/oauth/client-next.json"
}
~~~
{: #fig-active title="Example Active Document"}

## Establishing Reporting Configuration {#establishing}

A reporter establishes reporting configuration for a Client Identifier
URL when it accepts an active document that contains a
`client_metadata_report_uri` member and local policy permits reporting
for that client. A reporter has at most one reporting configuration for
each Client Identifier URL.

When the reporter accepts a later active document, it MUST replace the
reporting configuration with the values from that document, or discard
the reporting configuration if that document has no
`client_metadata_report_uri` member.

A reporter MUST NOT establish or change reporting configuration from an
active document that it does not accept, or from a candidate document.

If retrieval of the active document fails, or the reporter does not
accept a later active document, the reporter MAY use the established
reporting configuration to report either failure.

A reporter confirms an accepted active document when it retrieves the
Client Identifier URL and accepts the document obtained, including when
its processing under {{CIMD}} treats a 304 (Not Modified) response as
validating a cached document that it accepts. Using cached metadata
without retrieving the Client Identifier URL is not a confirmation. A
reporter MUST discard reporting configuration 7 days after its last
confirmation of an accepted active document for that Client Identifier
URL. This period is independent of how long the reporter caches client
metadata ({{Section 6.2 of CIMD}}), so that a failed retrieval after
cached metadata expires remains reportable.

A reporter records results for a client only while it has reporting
configuration for that client, and associates each result with the
collector URL in that configuration when it records the result. A
reporter that has never accepted a document for a Client Identifier URL
has no reporting configuration for it, so problems with that URL,
including failures of its first retrieval, are not reported.

# Evaluation {#evaluation}

## Checks and Outcomes {#checks}

An evaluation applies one or more checks. This specification defines
the following checks; {{iana-checks}} establishes a registry for
additional checks.

`retrieval`:
: Whether the reporter obtained the document ({{digest}}) by retrieving
  it as described in {{Section 6 of CIMD}}, within the size limit it
  applies ({{Section 10.11 of CIMD}}). A `retrieval` result that names a
  member describes retrieval of the resource that member identifies,
  such as the JWK Set at `jwks_uri`.

`document_structure`:
: Whether the document is a JSON object {{RFC8259}} whose content meets
  the requirements of {{Section 4 of CIMD}}, as modified for candidate
  documents by {{candidate}}. This includes the restrictions in
  {{Section 4.1 of CIMD}} and the syntax that each member's definition
  requires.

`supported_values`:
: Whether the reporter implements, for any client, the values of the
  members it processes, such as `grant_types` and `response_types`
  ({{Section 6 of CIMD}}) and `token_endpoint_auth_method`. For an
  array-valued member, the check fails if the reporter does not support
  one or more of its values, even when the reporter uses the remaining
  values.

`metadata_policy`:
: Whether the reporter permits the metadata for this client under
  policy beyond the requirements of {{CIMD}}, for example a requirement
  that clients be confidential, or a restriction of `redirect_uris` to
  the origin of the Client Identifier URL ({{Section 4.4 of CIMD}}).

Each check produces one of the following outcomes:

`compatible`:
: The reporter performed the check and found no problem.

`incompatible`:
: The reporter performed the check, and the document or the retrieval
  failed it.

`indeterminate`:
: The reporter did not complete the check; the reason code states why
  ({{reasons}}).

Whether the reporter accepts an active document is determined by its
processing under {{CIMD}}, not by the outcomes. A reporter can accept a
document for which some results are `incompatible`; the optional
`accepted` member of an active document report ({{format}}) can state
whether it did.

## Reason Codes {#reasons}

An `incompatible` or `indeterminate` result carries one of the
following reason codes, or one registered as described in
{{iana-reasons}}. The check with which each is used is given in
parentheses.

`unreachable`:
: (`retrieval`) DNS resolution or the connection failed, the request
  timed out, or the reporter declined to connect
  ({{Section 10.7 of CIMD}}).

`tls_error`:
: (`retrieval`) The TLS handshake or server certificate validation
  failed.

`redirect`:
: (`retrieval`) The response had a 3xx status code and did not validate
  a cached copy. Redirects are not followed.

`http_error`:
: (`retrieval`) The response had a status code that is neither 200 nor
  3xx.

`too_large`:
: (`retrieval`) The response exceeded the reporter's size limit.

`invalid_json`:
: (`document_structure`) The document is not a JSON object.

`client_id_mismatch`:
: (`document_structure`) The `client_id` member is absent or is not
  identical to the Client Identifier URL.

`invalid_value`:
: (`document_structure`) A member value is malformed or prohibited, for
  example by {{Section 4.1 of CIMD}}.

`unsupported_value`:
: (`supported_values`) The reporter does not support the member's value
  or, for an array-valued member, one or more of its values.

`policy_violation`:
: (`metadata_policy`) The reporter's policy does not permit the member's
  value or, if no member is named, the metadata.

`runtime_evidence_required`:
: (any check, with `indeterminate`) The check requires evidence that is
  available only while processing an OAuth request.

`not_evaluated`:
: (any check, with `indeterminate`) The reporter did not perform the
  check, by local policy or because an earlier check failed.

## Active Document Evaluation {#active}

A reporter evaluates the active document as part of its processing
under {{CIMD}}. This specification adds no retrieval and no processing
step; the reporter records the results of its existing processing as
outcomes of the checks in {{checks}}. Retrievals of resources that the
active document references, such as the JWK Set at `jwks_uri`, are part
of that processing whenever they occur. The reporter records their
failures as `retrieval` results that name the referencing member, in
the report for the digest of the document that references them.

## Candidate Document Evaluation {#candidate}

A reporter MAY evaluate the candidate document named by its reporting
configuration, for example after it establishes reporting configuration
that includes a candidate URL, or when the candidate URL changes.

The reporter retrieves the candidate document as it retrieves a Client
ID Metadata Document ({{Section 6 of CIMD}}), including the prohibition
on following redirects, and applies the same size limit
({{Section 10.11 of CIMD}}) and request forgery protections
({{Section 10.7 of CIMD}}). The reporter MAY use HTTP caching
{{RFC9111}}, including conditional requests, to avoid retrieving an
unchanged candidate document. {{Section 4.5 of CIMD}} applies to a
candidate document as it applies to a Client ID Metadata Document.

A candidate document describes the client identified by the Client
Identifier URL, not a separate client identified by the candidate URL.
Its `client_id` member MUST be identical, under simple string
comparison ({{Section 6.2.1 of RFC3986}}), to the Client Identifier
URL. For candidate documents, this requirement replaces the requirement
in {{Section 4 of CIMD}} that the `client_id` value match the URL used
to retrieve the document. A reporter reports a mismatch as the outcome
`incompatible` for the `document_structure` check, with the reason
`client_id_mismatch`.

A reporter evaluating a candidate document:

1. MUST NOT use the candidate document's metadata to process any OAuth
   request, replace or modify the active document's metadata, admit a
   client, issue credentials or tokens, or change an existing grant or
   consent, and MUST NOT store the candidate document where request
   processing reads client metadata.

2. MUST NOT retrieve, or deliver reports to, the URLs in the candidate
   document's `client_metadata_report_uri` or
   `client_metadata_candidate_uri` members. The reporter MAY check
   those values as it checks any other member.

3. MUST NOT perform an authorization flow, token request, or client
   authentication on behalf of the candidate, and MUST NOT replay any
   credential presented by a client instance.

4. MUST NOT use the result of a candidate evaluation when processing an
   active document, including an active document whose octets are
   identical to an evaluated candidate document.

Other than the URLs excluded by item 2, a reporter MAY retrieve URLs in
the candidate document that it would retrieve when processing OAuth
requests with the candidate as the client's metadata, such as
`jwks_uri`, subject to {{Section 10.7 of CIMD}}.

A check whose result depends on evidence available only while
processing an OAuth request produces the outcome `indeterminate` with
the reason `runtime_evidence_required`. For example, a reporter can
determine that it supports `private_key_jwt` as a
`token_endpoint_auth_method` value, but not that deployed client
instances hold a private key corresponding to a public key the
candidate document publishes.

The following is an example of a candidate document that would replace
the active document in {{fig-active}}. It adds a redirect URI on
another origin, adds the device authorization grant type, and changes
the client authentication method:

~~~ json
{
  "client_id": "https://client.example/oauth/client.json",
  "client_name": "Example Client",
  "redirect_uris": [
    "https://client.example/callback",
    "https://app.example.net/callback"
  ],
  "grant_types": [
    "authorization_code",
    "refresh_token",
    "urn:ietf:params:oauth:grant-type:device_code"
  ],
  "response_types": ["code"],
  "token_endpoint_auth_method": "private_key_jwt",
  "jwks_uri": "https://client.example/oauth/jwks.json",
  "client_metadata_report_uri":
    "https://client.example/oauth/reports"
}
~~~
{: #fig-candidate title="Example Candidate Document"}

## Document Digest {#digest}

A reporter obtains a document when it receives the complete content of a
200 response or, for a candidate document, when a 304 (Not Modified)
response ({{Section 15.4.5 of RFC9110}}) to a conditional request
validates a copy that it has cached. For an active document, whether a
304 response validates a cached copy is determined by the reporter's
processing under {{CIMD}}. A retrieval of a document that does not
obtain it is a failed retrieval; a failure to retrieve a resource that
the document references is not.

A report identifies the document it describes by a digest of the
document's octets, which are the content of the HTTP response after any
content coding ({{Section 8.4.1 of RFC9110}}) has been removed. The
digest is a JSON object with the following members:

`algorithm`:
: REQUIRED. JSON string containing a hash algorithm key from the "Hash
  Algorithms for HTTP Digest Fields" registry established by
  {{RFC9530}}. In this version of the report format, the value MUST be
  `"sha-256"`.

`value`:
: REQUIRED. JSON string containing the base64url encoding
  ({{Section 2 of RFC7515}}) of the hash of the document's octets.

A publisher determines which version of its document a report
describes by computing the digest of each version it has served
({{version-confusion}}).

# Reports {#reports}

## Report Format {#format}

A report is a JSON object {{RFC8259}} with the following members:

`version`:
: REQUIRED. JSON number containing the version of the report format,
  which is `1` for this specification.

`report_id`:
: REQUIRED. JSON string that identifies the report. The value MUST be
  unique across all reporters with high probability; a version 4 or
  version 7 UUID {{RFC9562}} meets this requirement. A reporter uses the
  same value when it retries delivery of a report and a different value
  for every other report.

`client_id`:
: REQUIRED. JSON string containing the Client Identifier URL.

`document_type`:
: REQUIRED. JSON string: `"active"` for a report about the active
  document, or `"candidate"` for a report about a candidate document.

`candidate_uri`:
: REQUIRED if `document_type` is `"candidate"`; otherwise, it MUST NOT
  be present. JSON string containing the candidate URL from which the
  reporter retrieved, or attempted to retrieve, the candidate document.

`document_digest`:
: REQUIRED if the reporter obtained the document ({{digest}});
  otherwise, it MUST NOT be present. JSON object containing the digest
  of the document.

`accepted`:
: OPTIONAL if `document_type` is `"active"` and `document_digest` is
  present; otherwise, it MUST NOT be present. JSON boolean: `true` if
  the reporter accepted the document at its most recent evaluation in
  the reporting period, and `false` otherwise.

`period_start`:
: REQUIRED. JSON string containing the start of the reporting period as
  a `date-time` value ({{Section 5.6 of RFC3339}}) in UTC.

`period_end`:
: REQUIRED. JSON string containing the end of the reporting period, in
  the same format.

`issuer`:
: OPTIONAL. JSON string containing the reporter's issuer identifier
  ({{Section 2 of RFC8414}}). Collectors cannot verify it ({{forged}}).

`results`:
: REQUIRED. JSON array of one or more result objects.

A result object has the following members:

`check`:
: REQUIRED. JSON string naming the check ({{checks}}).

`outcome`:
: REQUIRED. JSON string: `"compatible"`, `"incompatible"`, or
  `"indeterminate"`.

`reason`:
: REQUIRED if `outcome` is `"incompatible"` or `"indeterminate"`;
  otherwise, it MUST NOT be present. JSON string containing a reason
  code ({{reasons}}).

`member`:
: OPTIONAL. JSON string containing the name of the top-level member of
  the document to which the result applies ({{result-scope}}).

`values`:
: OPTIONAL if `member` is present; otherwise, it MUST NOT be present.
  JSON array of the values of the named member to which the result
  applies, for example the values of an array-valued member that the
  reporter does not support.

`http_status`:
: OPTIONAL if `reason` is `"http_error"`; otherwise, it MUST NOT be
  present. JSON number containing the HTTP status code received.

`count`:
: OPTIONAL. JSON number containing the number of times the reporter
  observed the result during the reporting period. If present, it MUST
  be a positive integer.

A collector MUST ignore a report whose `version` it does not support,
and MUST ignore members it does not understand. A `check` or `reason`
value that a collector does not recognize does not make the report
invalid.

## Result Scope {#result-scope}

A result that names a member applies only to that member or, for the
`retrieval` check, to the resource that member identifies. A result that
names no member applies to the document as a whole. For the `retrieval`
check, it describes retrieval of the document itself. For other checks,
an `incompatible` or `indeterminate` result that names no member
describes a problem without identifying a member, and a `compatible`
result that names no member states that the check found no problem other
than those reported in results that name members.

## Active Document Reports {#active-reports}

An active document report aggregates the results that a reporter
observed during a reporting period. It contains results for one Client
Identifier URL, collector URL ({{establishing}}), reporting period, and
document digest. A reporter MUST report results for different document
digests in separate reports, and MUST report results for failed
retrievals ({{digest}}) in reports that have no digest.

Each distinct combination of check, member, values, outcome, reason,
and HTTP status code appears as one result. Counts of different results
are not additive, because one evaluation produces a result for each
check it performs.

A reporter MAY omit results with the outcome `compatible`, and delivers
no report if no results remain. A check that does not appear in an
active document report conveys no information about whether the
reporter performed it.

## Candidate Reports {#candidate-reports}

A candidate report describes one evaluation of a candidate document. It
contains at least one result for each check defined in this
specification, including results with the outcome `compatible`. A
reporter that does not perform one of those checks, by local policy or
because an earlier check failed, reports it with the outcome
`indeterminate` and the reason `not_evaluated`. For example, after a
failed retrieval, the `document_structure`, `supported_values`, and
`metadata_policy` checks are reported as `not_evaluated`. A reporter MAY
also report results for checks defined elsewhere.

## Report Content Restrictions {#content}

Reports MUST NOT include access tokens, refresh tokens, authorization
codes, client assertions or other credentials, private keys, the
content of OAuth requests or responses, or identifiers of end users or
client instances. Reporters SHOULD convey diagnostic information
through registered check and reason values rather than free-form text.

## Examples {#examples}

The following example shows a report of an evaluation of the candidate
document in {{fig-candidate}}. The reporter does not support the device
authorization grant, and its policy restricts redirect URIs to the
origin of the Client Identifier URL. The `compatible` results that name
no member state that those checks found no other problem. The digest is
computed over the octets of the document exactly as shown in
{{fig-candidate}}, with LF line endings and no trailing line break.

~~~ json
{
  "version": 1,
  "report_id": "4c4175e6-2070-40c1-9015-184a1523bcba",
  "client_id": "https://client.example/oauth/client.json",
  "document_type": "candidate",
  "candidate_uri": "https://client.example/oauth/client-next.json",
  "document_digest": {
    "algorithm": "sha-256",
    "value": "_HhRKZxuBzacmpnLQL2CHaJsd8mWZd0wN3kNGZhc7-w"
  },
  "period_start": "2026-11-09T12:04:10Z",
  "period_end": "2026-11-09T12:04:11Z",
  "issuer": "https://as.example",
  "results": [
    {"check": "retrieval", "outcome": "compatible"},
    {"check": "document_structure", "outcome": "compatible"},
    {
      "check": "supported_values",
      "member": "grant_types",
      "values": ["urn:ietf:params:oauth:grant-type:device_code"],
      "outcome": "incompatible",
      "reason": "unsupported_value"
    },
    {"check": "supported_values", "outcome": "compatible"},
    {
      "check": "metadata_policy",
      "member": "redirect_uris",
      "values": ["https://app.example.net/callback"],
      "outcome": "incompatible",
      "reason": "policy_violation"
    },
    {"check": "metadata_policy", "outcome": "compatible"}
  ]
}
~~~

The following example shows an active document report covering one
hour in which every retrieval of the Client Identifier URL received a
403 response. No document was obtained, so the report has no digest.
The reporter omits its issuer identifier.

~~~ json
{
  "version": 1,
  "report_id": "0f9a3c1e-5b7d-4e21-9a4c-7d2b8e6f1a30",
  "client_id": "https://client.example/oauth/client.json",
  "document_type": "active",
  "period_start": "2026-11-09T12:00:00Z",
  "period_end": "2026-11-09T13:00:00Z",
  "results": [
    {
      "check": "retrieval",
      "outcome": "incompatible",
      "reason": "http_error",
      "http_status": 403,
      "count": 12
    }
  ]
}
~~~

# Report Delivery {#delivery}

## Delivery Request {#request}

A reporter delivers a report by sending an HTTP POST request to the
collector URL associated with the report's results ({{establishing}}),
with the report as the request content and a `Content-Type` header
field of `application/oauth-client-metadata-report+json`. Each request
carries one report. The reporter delivers a report to that collector
URL even if the reporting configuration has since been replaced or
discarded.

The reporter MUST NOT include cookies, HTTP authentication
credentials, or OAuth credentials in the request, and MUST NOT copy
header fields or content from any OAuth request into it.

The following is an example of a delivery request; the content is a
report such as those in {{examples}}:

~~~ http-message
POST /oauth/reports HTTP/1.1
Host: client.example
Content-Type: application/oauth-client-metadata-report+json

{"version":1,"report_id":"4c4175e6-2070-40c1-9015-184a1523bcba",...}
~~~

## Response Handling {#response}

A 2xx status code indicates that the collector received the report. A
collector SHOULD respond with a 2xx status code to a repeated delivery
of a report it has already received, which it detects by `report_id`,
and SHOULD respond with a 4xx status code to a report it rejects. A
collector can discard reports whose `client_id` identifies a client for
which it does not collect reports.

The reporter MUST NOT follow redirects; a 3xx status code is a delivery
failure that the reporter does not retry. After a connection failure, a
timeout, a 429 (Too Many Requests) status code
({{Section 4 of RFC6585}}), or a 5xx status code, the reporter MAY retry
delivery within the time limit in {{bounds}}. A reporter that retries
SHOULD increase the interval between attempts and SHOULD honor the
`Retry-After` header field ({{Section 10.2.3 of RFC9110}}). The reporter
SHOULD NOT retry after any other 4xx status code.

## Limits {#bounds}

A report MUST NOT exceed 16384 octets. A reporter whose active document
results do not fit in one report MAY divide them among several active
document reports that have the same `document_digest` and reporting
period. Collectors SHOULD support reports up to that size.

A reporter MUST NOT attempt delivery of a report more than 24 hours
after its `period_end`.

A reporter SHOULD use reporting periods of at least one hour for active
document reports. A reporter SHOULD NOT report on the same candidate
document digest, or on failed retrievals of the same candidate URL,
more than once in any 24-hour period unless the results change, for
example because the reporter's supported values or policy changed.


For each origin, a reporter SHOULD limit the rate of delivery requests,
the rate of candidate retrievals, and the number of Client Identifier
URLs for which it maintains reporting configuration. A reporter can
apportion a per-origin delivery limit among the collector URLs on that
origin ({{shared-origins}}). A reporter MAY sample, aggregate, delay,
or discard reports, and MAY discard evaluations or reporting
configuration, under local policy, for example when it reaches a
resource limit.

# Interpreting Reports {#interpreting}

A report describes the named checks, applied by one reporter to the
identified octets, during the reporting period. It describes no other
reporter, document, or time.

The absence of reports can result from the absence of problems, from
authorization servers that do not implement this specification or do
not report for the client, from sampling or suppression, from delivery
failure, or from reporting configuration never having been established.
The absence of reports is not approval.

No outcome, and no combination of outcomes, states that a document is
safe to deploy. In particular, a candidate report whose results are all
`compatible` does not establish:

* that client instances are ready for the change, for example that
  they hold keys in a new JWK Set or handle a new redirect URI;

* that authorization servers other than the reporter would accept the
  document once it is served at the Client Identifier URL;

* that the reporter would reach the same results after its supported
  values or policy change; or

* that serving the document will not cause the reporter to invalidate
  grants or require new consent, which {{Section 10.5 of CIMD}} permits
  when metadata changes.

The document digest identifies the document, not every input to its
evaluation. Results that depend on referenced resources, such as the
JWK Set at `jwks_uri`, can differ between evaluations of documents with
the same digest, and a candidate report does not establish that those
resources will remain available or unchanged.

An `accepted` value of `true` in an active document report describes
the reporter's past processing; it does not establish that the reporter
will continue to accept the document.

Reporting does not synchronize caches. A reporter can continue to use
cached client metadata after the publisher serves a new document
({{Section 6.2 of CIMD}}), so active document reports for a previous
digest can arrive after a change.

# Security Considerations {#security}

The security considerations in {{Section 10 of CIMD}} apply to this
specification.

## Reporting and Authorization Decisions {#sec-independence}

Because a reporter decides OAuth requests independently of reporting
({{independence}}), an attacker that blocks the collector, serves a
hostile candidate document, or delivers forged reports to a collector
does not affect authorization outcomes.

## Request Amplification {#amplification}

Any party can cause an authorization server to retrieve a Client
Identifier URL by sending an OAuth request that names it. Because both
members defined in {{metadata}} are restricted to the origin of the
Client Identifier URL, candidate retrieval and report delivery are
directed only at the origin that the authorization server already
contacts for that client. The only other requests that reporting can
cause are retrievals of resources a candidate document references, such
as a JWK Set, which an active document could equally reference.
Because reporters do not follow redirects, a collector cannot direct
reports to another origin.

Aggregation by document digest and reporting period bounds the number
of reports independently of the number of OAuth requests. An attacker
can create any number of Client Identifier URLs on an origin it
controls; the per-origin limits in {{bounds}} bound the reporting
configurations, candidate retrievals, and delivery requests that those
URLs can cause. The size limit of {{Section 10.11 of CIMD}} and
asynchronous processing limit the cost of each candidate evaluation.

## Shared Origins {#shared-origins}

An origin can host documents for several publishers, for example on a
hosting platform or a CIMD Service ({{Section 10.13 of CIMD}}). On such
an origin, the same origin does not imply the same publisher: one
publisher can name a URL controlled by another publisher as its
collector, and reporters will deliver reports that the recipient did
not request to that URL. Such reports concern only the naming
publisher's client. Per-origin limits ({{bounds}}) bound this traffic
across collector URLs that differ only in path, but they also let one
publisher consume the delivery budget of others on the same origin,
which reporters can mitigate by apportioning the limit among collector
URLs. A publisher that names another publisher's document as its
candidate receives a `client_id_mismatch` result, and candidate
evaluation affects no client ({{candidate}}). Operators of shared
origins can restrict the values that publishers use for these members,
for example to URLs under the path of the publisher's own document. A
mechanism by which collectors authorize reports, such as the external
destination verification of DMARC ({{Section 4 of RFC9990}}), would
address this as well.

## Forged Reports {#forged}

HTTPS authenticates the collector to the reporter, but not the reporter
to the collector. Any party can deliver a report to a collector, with
any `issuer` value. Unless reporters are authenticated by other means,
for example with HTTP Message Signatures {{RFC9421}} and keys that the
publisher trusts, publishers should treat reports as unverified
diagnostic input and should not make security-relevant changes, such as
changes to keys, redirect URIs, or client authentication methods,
automatically in response to a report. An attacker could otherwise
forge reports to induce a publisher to revert a change or to adopt a
weaker configuration. Collectors can apply size and rate limits to
delivery requests.

## Candidate Evaluation as a Policy Oracle {#oracle}

Any party that controls a Client Identifier URL can publish candidate
documents and learn from the resulting reports which values and
policies a reporter permits, without involving an end user. Much of
this information is also available from authorization server metadata
{{RFC8414}} or by sending OAuth requests. A reporter controls what it
discloses through the granularity of its results, for example by
reporting `policy_violation` without naming a member or listing
values, and MAY limit
candidate evaluation to clients that meet local policy, such as clients
for which it has already processed OAuth requests.

## Document Version Confusion {#version-confusion}

A publisher can attribute a report to the wrong version of its
document, particularly while reporters hold an earlier version in
cache. The document digest binds each result to the exact octets of the
document, though not to the resources the document references
({{interpreting}}), and publishers should match reports to versions by
digest rather than by time of arrival. A digest that matches no version
the publisher served can indicate that an intermediary modified the
document or that different reporters receive different content. A
`compatible` result for a candidate document applies only to the
evaluated octets; any change, including a change to whitespace,
produces a different digest and therefore a different document.

# Privacy Considerations {#privacy}

The privacy considerations in {{Section 11 of CIMD}} apply to this
specification.

## Disclosure of Deployment Relationships {#relationships}

A report informs the publisher that the reporter has encountered the
client. The `issuer` member makes the reporter's identity explicit, and
a reporter that does not want to disclose its identity omits it. On a
platform that hosts authorization servers for several tenants, each
tenant's authorization server is a separate reporter whose issuer
identifier can identify the tenant, so including it discloses the
tenant's relationship with the client. The network source of a delivery
request remains visible to the collector and can identify the
reporter's operator. A reporter that does not want to disclose a
relationship with a client does not report for that client.

## End-User Activity {#activity}

Retrieval of a Client Identifier URL can reveal when users authorize a
client ({{Section 11.1 of CIMD}}). Active document reports aggregated
over a reporting period add little to that information, although the
`count` member reveals the number of retrievals in the period. At a
reporter with few users, a report delivered immediately after a
retrieval can reveal the timing of one user's activity; such reporters
can use longer reporting periods or delay delivery.

Candidate reports carry the same risk in a different form: a candidate
report describes one evaluation, so if a reporter evaluates a candidate
document immediately after an OAuth request causes it to retrieve the
active document, the report's reporting period reveals when that
request occurred. Reporters can evaluate candidate documents on a
schedule independent of OAuth requests, or delay candidate reports.
Reports carry no end-user identifiers ({{content}}).

# IANA Considerations {#iana}

## OAuth Dynamic Client Registration Metadata Registry {#iana-dcr}

This specification requests registration of the following client
metadata in the "OAuth Dynamic Client Registration Metadata" registry
{{IANA.oauth-parameters}} established by {{RFC7591}}:

Client Metadata Name:
: `client_metadata_report_uri`

Client Metadata Description:
: URL of the collector that receives client metadata reports

Change Controller:
: IETF

Specification Document(s):
: {{metadata}} of this specification

Client Metadata Name:
: `client_metadata_candidate_uri`

Client Metadata Description:
: URL of a candidate client metadata document to be evaluated

Change Controller:
: IETF

Specification Document(s):
: {{metadata}} of this specification

## Media Type Registration {#iana-media-type}

This specification requests registration of the following media type
in the "Media Types" registry {{IANA.media-types}} in the manner
described in {{RFC6838}}:

Type name:
: application

Subtype name:
: oauth-client-metadata-report+json

Required parameters:
: N/A

Optional parameters:
: N/A

Encoding considerations:
: binary; a report is a JSON object {{RFC8259}}, which is encoded in
  UTF-8.

Security considerations:
: See {{security}} of this specification.

Interoperability considerations:
: N/A

Published specification:
: This specification

Applications that use this media type:
: OAuth authorization servers that deliver client metadata reports, and
  the collectors that receive them.

Fragment identifier considerations:
: N/A

Additional information:
: Deprecated alias names for this type: N/A

  Magic number(s): N/A

  File extension(s): N/A

  Macintosh file type code(s): N/A

Person & email address to contact for further information:
: Karl McGuinness, public@karlmcguinness.com

Intended usage:
: COMMON

Restrictions on usage:
: none

Author:
: Karl McGuinness

Change controller:
: IETF

## Registration Procedures {#iana-procedures}

The registries established in {{iana-checks}} and {{iana-reasons}} are
in the "OAuth Parameters" registry group {{IANA.oauth-parameters}}.
Values are registered via Specification Required ({{Section 4.6 of RFC8126}}). Registration requests should be sent to
<oauth-ext-review@ietf.org> to initiate a two-week review period.
However, to allow for the allocation of values prior to publication of
the final version of a specification, the designated experts may
approve registration once they are satisfied that the specification
will be completed and published.

Designated experts ({{Section 5 of RFC8126}}) should verify that a
proposed value is unique, has a bounded meaning that fits the purpose
of the registry, and does not carry free-form data or information that
identifies end users or client instances.

## OAuth Client Metadata Report Checks Registry {#iana-checks}

This specification establishes the "OAuth Client Metadata Report
Checks" registry.

### Registration Template {#iana-checks-template}

Check Name:
: The check identifier, as used in the `check` member of a result
  object.

Check Description:
: A brief description of what the check evaluates.

Change Controller:
: For IETF Stream RFCs, "IETF". Otherwise, the name of the responsible
  party.

Specification Document(s):
: A reference to the document that defines the check.

### Initial Registry Contents {#iana-checks-contents}

For each of the following entries, the Change Controller is "IETF" and
the Specification Document is {{checks}} of this specification.

| Check Name | Check Description |
|---|---|
| `retrieval` | Whether the document, or a resource it references, could be retrieved |
| `document_structure` | Whether the document meets the format and content requirements of a Client ID Metadata Document |
| `supported_values` | Whether the reporter implements the member values the document uses |
| `metadata_policy` | Whether the reporter's policy permits the metadata for the client |
{: #tab-checks title="Initial Checks"}

## OAuth Client Metadata Report Reasons Registry {#iana-reasons}

This specification establishes the "OAuth Client Metadata Report
Reasons" registry.

### Registration Template {#iana-reasons-template}

Reason Name:
: The reason code, as used in the `reason` member of a result object.

Applicable Checks:
: The checks with which the reason code is used, or "any".

Reason Description:
: A brief description of the condition the reason code reports.

Change Controller:
: For IETF Stream RFCs, "IETF". Otherwise, the name of the responsible
  party.

Specification Document(s):
: A reference to the document that defines the reason code.

### Initial Registry Contents {#iana-reasons-contents}

For each of the following entries, the Change Controller is "IETF" and
the Specification Document is {{reasons}} of this specification.

| Reason Name | Applicable Checks | Reason Description |
|---|---|---|
| `unreachable` | `retrieval` | Connection failed or timed out |
| `tls_error` | `retrieval` | TLS or certificate failure |
| `redirect` | `retrieval` | 3xx response, not followed |
| `http_error` | `retrieval` | Status neither 200 nor 3xx |
| `too_large` | `retrieval` | Size limit exceeded |
| `invalid_json` | `document_structure` | Not a JSON object |
| `client_id_mismatch` | `document_structure` | `client_id` mismatch |
| `invalid_value` | `document_structure` | Malformed or prohibited value |
| `unsupported_value` | `supported_values` | Value not supported |
| `policy_violation` | `metadata_policy` | Not permitted by policy |
| `runtime_evidence_required` | any | Needs request-time evidence |
| `not_evaluated` | any | Check not performed |
{: #tab-reasons title="Initial Reason Codes"}

--- back

# Document History {#history}
{:removeinrfc}

-00

* Initial version.
