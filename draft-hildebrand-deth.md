---
title: DNS Editing Through HTTPS (DETH)
abbrev: I-D
docname: draft-hildebrand-deth-00
category: info
ipr: trust200902

author:
 -
    ins: J. Hildebrand
    name: Joe Hildebrand
    organization: Cisco Systems
    email: jhildebr@cisco.com
 -
    ins: P. Hoffman
    name: Paul Hoffman
    organization: ICANN
    email: paul.hoffman@icann.org

normative:
  RFC2119:
  RFC3597:
  RFC4035:
  RFC6125:
  RFC6749:
  RFC7230:
  RFC7235:

informative:
  I-D.greevenbosch-appsawg-cbor-cddl:
  I-D.draft-ietf-appsawg-http-problem-03:
  draft-jennings-app-dns-update:
    title: HTTP API for Updating DNS Records
    author:
      -
        ins: C. Jennings
        name: Cullen Jennings
      -
        ins: T. Daly
        name: Tom Daly
      -
        ins: J. Hitchcock
        name: Jeremy Hitchcock
    target: https://tools.ietf.org/html/draft-jennings-app-dns-update
    date: 2009
  RFC3007:
  dig:
    title: dig utility
    author:
      org: ISC
    target: https://www.isc.org/downloads/bind/
    date: 2016
  curl:
    title: curl program
    author:
      ins: D. Stenberg
      name: Daniel Stenberg
    target: https://curl.haxx.se/
    date: 2016
  IANA-rrtypes:
    title: Resource Record TYPEs
    target: http://www.iana.org/assignments/dns-parameters/dns-parameters.xhtml#dns-parameters-4
    author:
      org: IANA
    date: 2016-02-23

--- abstract

There is a strong desire in many communities for service operators
to be able to dynamically update DNS records
in an easy-to-deploy, standardized method. For example, operating SIP requires
DNS records to be added and updated as the SIP service starts and is moved
among different servers. This document describes an HTTPS-based mechanism
for service operators who are authorized by their DNS administrator to add,
change, and delete DNS records.

--- middle

<!--

Notes between Joe and Paul go here

-->

# Introduction

This document describes a standardized mechanism called DNS Editing Through
HTTPS (DETH) for developer-friendly dynamic DNS updates over HTTPS {{RFC7230}}.
Such a mechanism allows a DNS administrator to authorize particular users to
update the records in their zones without any manual intervention on the part of
the DNS administrator.

All transactions described here are authorized. Two types of authorization are
specified: authenticated by HTTP Digest {{RFC7235}} and OAUTH {{RFC6749}}. The
DNS administrator can create policies to allow different users different
capabilities for updating zones; such policies are outside the scope of this
document.

A client determines the DETH server for a zone using a DNS lookup. It then
sends HTTPS queries to that server to get information about what the client
can change, as well as requests for changes.

The DETH protocol is meant for users who do not currently have a
way to update a zone using the DNS protocol itself.
Although there already is a protocol for dynamic update of DNS records,
{{RFC3007}}, it is rarely used in practice for more anything more complicated
than inserting a single A or AAAA record. In many scenarios,
using DETH is a simpler way to allow users to update a zone than provisioning
DNS dynamic update.

Some large DNS
operators have implemented their own non-standard mechanisms for allowing
users to update their DNS records, often using HTTP. This indicates that
HTTP-based update is desired in the industry, and a standardized mechanism
could be valuable in many environments.

## Goals of this Protocol

 * Authorized additions of a new record to a zone

 * Authorized changes to the RDATA and TTL of a record in a zone

 * Authorized deletions of a record from a zone

 * Discovery of a URI used for editing a zone, such that the client doing the
   editing can tell that the server they are contacting is authorized for the
   record being edited

 * Discovery of what actions can be taken by an authorized user

 * Responses can give useful information about why a request was rejected

 * Ease of writing an implementation for a wide variety of languages and
   platforms is of paramount importance

## Non-Goals for this Protocol

 * Updating the contents of any type of configuration other than DNS zones

 * Managing DNS servers for anything other than the contents of zones for
   which they are authoritative, such as causing them to reload a zone after
   update or to clear cache entries

 * Pluggable authorization modules

 * Editing records in DNS Classes other than IN

## Possible Deployment Architectures

There are many ways that the DETH protocol could be deployed. This section gives
some sample architectures.

Native:
: An authoritative DNS server system also speaks DETH and uses the results of
updates directly in the zones it serves. The DETH protocol could be built
into DNS server software.

Bridge:
: A DETH server is deployed in front of the management interface for an
authoritative DNS server. The DETH server receives authorized updates from
users, and uses DNS dynamic update {{RFC3007}} to update the zones on the
authoritative DNS server.

Data store front end:
: A DETH server receives authorized updates from users, modifies a DNS data
store, tells DNS server to reload.

## Actors

The following actors are used in this document:

DETH Server:
: A server that speaks HTTPS, and provides an implementation of the server
side of the DETH API, rooted at a discoverable HTTPS URI.  The DETH Server
is authorized by the associated DNS infrastructure for a set of domains to
make policy decisions about DNS edits.  The DETH server is responsible for
enforcing authorization in conformance with this policy.

DETH Client:
: A piece of authorized software that would like to make changes to one or
more DNS records.  In order to make man-in-the-middle attacks more difficult,
the DETH client is responsible for ensuring that it is communicating with the correct
DETH Server for the domain it wants to modify.

Parent Domain:
: The thing that you want to add records to.  (Paul: need DNS terminology here)

## Terminology

In this document, the key words "MUST", "MUST NOT", "REQUIRED",
"SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY",
and "OPTIONAL" are to be interpreted as described in BCP 14, RFC 2119
{{RFC2119}}.

# Protocol

The DETH protocol uses a simple client-server interactions. The client determines
the location of the server either through a DNS lookup or local configuration.
After that, all protocol interactions are over HTTPS.

In the following sections, the client wants to edit records in the "example.com"
zone. The HTTPS examples use the "dig" program {{dig}} and "curl" program {{curl}} to get URLs,
but clients will most likely use internal calls to get this information.

## Determining the DETH Server for a Zone

A client uses a DNS query with the TXT RTYPE using the name of the parent
zone, prefixed with the "_deth" label, for the QNAME. The answer is the base URI for
the HTTPS server running the DETH protocol. This URI returned MUST use the HTTPS
scheme.

For example, to find the DETH server for the "example.com" zone, the query
would have a QNAME of "_deth.example.com" and a QTYPE of TXT. A command-line
equivalent would be:

~~~ shell
$ dig +short TXT _deth.example.com
https://example.com/deth/v1/
~~~

If no TXT record is found for a name, the client can assume that the parent
does not implement the DETH protocol.  However, explicit configuration might
still allow a client to find a DETH server.

If the DETH server is found using the DNS lookup described here, the client MUST perform the
following checks before using the result:

 * Ensure that the string is a valid HTTPS URI (see {{RFC7230}}, section 2.7.2).
 * Ensure that the TXT record has a valid DNSSEC {{RFC4035}} signature OR that the
   host portion of the URI matches the parent domain, with no port specified.
 * Ensure that the certificate offered when the URI is accessed using HTTPS
   matches the domain name using the rules in {{RFC6125}}.
 * Ensure that the DETH server's certificate is signed by a trusted Certificate
   Authority.

## Determining Authorized Edits {#directory}

The DETH client does an HTTPS GET request to the DETH server go get a list of
edits that the client is authorized to perform. For example:

~~~ shell
curl -X GET https://example.com/deth/v1/
~~~

Might return:

~~~ json
{::include dir.json}
~~~
{: #example-dir title="Directory Example"}

The valid keys for the top level JSON object are a popular subset of
{{IANA-rrtypes}}, plus a mechanism for supporting all other RDATA.
The response is formally described in CDDL
{{I-D.greevenbosch-appsawg-cbor-cddl}} as:

~~~ cddl
{::include deth-dir.cddl}
~~~
{: #directory-cddl title="DETH Directory CDDL"}

# Authentication and Authorization

All requests are authenticated either by by HTTP Digest {{RFC7235}} and OAUTH {{RFC6749}}.
Parameters supported for either of these mechanisms are determined by the DETH Server.
This document makes no recommendations for best authentication practices beyond
what have already been described in other documents published by the IETF.

After the DETH server authenticates a user, it determines which actions that user
is authorized to make. If using HTTP Digest, the authorization policy probably
comes from a database. If using OAUTH, that determination might be part of the
OAUTH interaction.

# Forming Request URIs {#formingURL}

When a client wants to edit a particular DNS record, it appends the full name of
the record to the URI for the RTYPE found in the directory JSON (see
{{directory}}).  For example, if the directory JSON was that specified in
{{example-dir}}, and the client wanted to edit a `AAAA` record for
`foo.example.com`, the URL would be

~~~ shell
https://example.com/deth/v1/AAAA/foo.example.com
~~~

TODO: specify more rules about URL combination to avoid attacks.

# Record Editing

This section describes the semantics of requests to edit DNS
records.  The specification covers how to specify which
edits are desired, but does not yet cover how the DNS server deals with
updating SOA records, nor how any DNSSEC records would need to be updated.

## Encoding in JSON

The JSON sent to the URIs formed according to the rules in {{formingURL}} looks
like:

~~~ json
{::include update.json}
~~~
{: #example-update title="Update Example"}

All of the potential updates are specified by the following CDDL:

~~~ cddl
{::include deth-update.cddl}
~~~
{: #update-cddl title="DETH Update CDDL"}

For updates that match the rrtype_update syntax, the rules for encoding RDATA
from {{RFC3597}} are used.

## Getting Records

Sending `GET` requests to the URIs formed in {{formingURL}} is
supported in order to allow clients more easily edit records.

TODO: The response
types for `GET` responses will be specified in a future version of this
document.

## Creating Records {#create}

A new record is created by sending the desired JSON document that matches
{{update-cddl}} to the URI formed by the rules in {{formingURL}}, using the
`POST` HTTP verb.

If a TTL is not sent with the request, a system default will be used.  The
response from this `PUT` will be the JSON form of the record, as inserted.
This response MUST have the TTL included.

## Deleting Records {#delete}

A new record is created by sending the desired JSON document that matches
{{update-cddl}} to the URI formed by the rules in {{formingURL}}, using the
`DELETE` HTTP verb.

Currently, this will delete all matching records.  TODO: Once matching rules have been
defined in a later version of this document, individual record deleting may be
allowed.

## Updating Records

Updating records in place is not yet specified.  A prerequisite for this
feature will be a way to match existing records, so that only one of several
existing records with the same name and RTYPE will be modified.

For now, a combination of request as specified in {{delete}} and {{create}}
may be used.

TODO: Add matching feature or change this section.

##  Return Codes and Errors

Errors use the approach from {{I-D.ietf-appsawg-http-problem}}.

TODO: Error information will be specified in a future revision of this document.

# IANA Considerations

No new IANA registries are expected.

# Security Considerations

Careful authorization of all edits is very important. All changes that are
allowed by this specification MUST be authorized using the model described.

--- back

# Acknowledgements

This document borrows heavily from many earlier protocols.
Some of the text of this document is liberally lifted from
the long-expired {{draft-jennings-app-dns-update}}.
