---
stand_alone: true
ipr: trust200902
docname: draft-ietf-lpwan-coap-static-context-hc-latest
cat: info
pi:
  symrefs: 'yes'
  sortrefs: 'yes'
  strict: 'yes'
  compact: 'yes'
title: 6LPWA Static Context Header Compression (SCHC) for CoAP
abbrev: LPWAN CoAP compression
wg: lpwan Working Group
date: 2016-12
author:
- ins: A. Minaburo
  name: Ana Minaburo
  org: Acklio
  street: 2bis rue de la Chataigneraie
  city: 35510 Cesson-Sevigne Cedex
  country: France
  email: ana@ackl.io
- ins: L. Toutain
  name: Laurent Toutain
  org: Institut MINES TELECOM ; TELECOM Bretagne
  street:
  - 2 rue de la Chataigneraie
  - CS 17607
  city: 35576 Cesson-Sevigne Cedex
  country: France
  email: Laurent.Toutain@telecom-bretagne.eu
normative:
  rfc4944:
  rfc3095:
  rfc4997:
  rfc5225:
  rfc6282:
  rfc7252:
  rfc7967:
  rfc1332:
  I-D.toutain-lpwan-ipv6-static-context-hc:

--- abstract


This draft discusses the way SCHC can be applied to CoAP headers and extend
the number of  functions (CDF) to optimize compression.

--- middle

# Introduction {#Introduction}

{{I-D.toutain-lpwan-ipv6-static-context-hc}} defines a compression
technique for LPWA network based on static context.
This context is said static since the element values composing the context
are not learned during packet exchanges but previously installed.
The context is known by both ends.
A context is composed of a set of rules (referenced by rule ids). A rule
describes the header fields with some associated Target Values (TV). A Matching
Operator (MO) is associated to each field. The rule is selected if all the
MO matches . A Compression Decompression Function is associated to each field
to define the link between the compressed and decompressed value for a specific
field.

This draft discusses the way SCHC can be applied to CoAP headers and extend
the number of  functions (CDF) to optimize compression.


# Compressing CoAP

CoAP {{RFC7252}} is an implementation of a the REST architecture for contrained devices. Gateway
between CoAP and HTTP can be easily build since both protocol  uses the same
address space (URL), caching mechanisms and methods.

Nevertheless, if limited, the size of a CoAP header may be incompatible with
LPWAN constraints and some compression may be needed to reduce the header
size. CoAP compression is not straightforward. Some differences between IPv6/UDP
and CoAP can be enlighten. CoAP differs from IPv6 and UDP protocols:

* IPv6 and UDP are symmetrical protocols. The same fields are found in the
  request and in
  the answer, only location in the header may change (e.g. source and destination
  fields). A CoAP request is different from  an answer. For instance, the URI-path
  option is mandatory in the request and may not be found in the response.

* CoAP also obeys to the client/server paradigm and the compression rate can
  be different if the request is issued from a LPWAN node or from an external
  device. For instance in the former case the token size may be set to one
  byte. In the latter case, the token size cannot be constraint and be up to
  15 byte long.

* In IPv6, main header and UDP fields have a fixed size. In CoAP, Token size
  may vary from 0 to 15 bytes, length is given by a field in the header. More
  systematically, the options are described using the Type-Length-Value principle.
  Evenmore regarding the option size value, the coding will be different.

* options type in CoAP are not defined with the same value. The Delta TLV coding
  makes that the type is not independant of previous option and may vary regarding
  the options contained in the header.


## CoAP usages

A LPWAN node can either be a client or a server and sometimes both.
In the client mode, the LPWAN node sends request to a server and expected
answer or acknowledgements. Acknowledgements can be at 2 different levels:

* transport level, a CON message is acknowledged by an ACK message. NON confirmable
  messages are not acknowledged.

* REST level, a REST request is acknowledged by an "error"
  code. {{RFC7967}} defines an option to limit the number of
  acknowledgements.


Note that acknowledgement can be optimized and a REST level acknowledgement
can be used as a transport level acknowledgement.


## CoAP protocol analysis

CoAP defines the following fields:

* version (2 bits): this field can be elided during a compresssion

* type (2 bits): defines the type of the transport messages, 4 values are defined.
  Regarding the type of exchange, if only NON messages are sent or CON/ACK
  messages, this field can be reduced to 0 or 1 bit.

* token length (4 bytes). The standard allows up to 15 bytes for a token length.
  If a fix token size is chosen, then this field can be elided. If some variation
  in length are needed then 1 or 2 bits could be enough for most of LPWAN applications.

* code (8 bits). This field codes the request and the response values. CoAP
  represents in a more compact way, coding used in HTTP, but the coding is
  not optimal.

* message id (16 bits). This value is used to acknowledge CON
  frames. The size of this field is computed to allow the anticipation
  (how many frames can be sent without acknowledgement). When a value
  is used, {{RFC7252}} defines the time before it can be reused
  without ambiguities. This size may be too large for a LPWAN node
  sending or receiving few messages a day.

* Token (0 to 15 bytes). Token identifies active flows. Regarding the usage
  (stability of in time and limited number), a short token (1 Byte) can be
  enough.

* options are coded through delta-TLV. The delta-T depends of previous values,
  length is encoded inside the option. {{RFC7252}} distinguishes
  repeatable options that can appear several time in the header.
  Among them we can distinguish:

  * list options which appear several time in the header but are exclusive such
    as the Accept option.

  * cumulative options which appear several time in the header but are part of
    a more generic value such as Uri-Path and Uri-Query.

  For a given flow some value options are stable through time. Observe,
  ETag, If-Match, If-None-Match and Size varies in each message. Options
  can be stored in a SCHC context and cumulative options can be stored
  globally.


The CoAP protocol must not be altered by the compression/decompression phase,
but if no semantic is attributed to a value, it may be changed during this
phase. For instance the compression phase may reduce the size of a token
but must maintain its unicity. The decompressor will not be able to restore
the original value but behavior will remain the same. If no special semantic
is assigned to the token, this will be transparent. If a special semantic
is assigned to the token, its compression may not be possible.

This implies that the compressor/decompressor must be aware of the protocol
state machine and do not processes request and response the same way.

A conservative compression leaves the field value unchanged. Non conservative
compression can be used when a CoAP implementation has not been defined to
work specifically with LPWAN network and uses large values for fields.

### CoAP Compression Decompression Function

To compress more efficiently CoAP message, several Compression/Decompression
Function (CDF) must be defined.

#### Static-mapping

The goal of static-mapping is to reduce the size of a field by allocating
shorter value. The mapping is known by both ends and stored in a table in
both end context. The Static-mapping is conservative.

Static-mapping may be applied to several fields. For instance the type field
may be reduced from 2 bits to 1 bit if only CON/ACK type is used, but the
main benefit is  compressing the code field.

The CoAP code field defines a tricky way to ensure compatibility with HTTP
values. Nevertheless only 21 values are defined by {{RFC7252}} compared to the 255 possible values. So it could efficiently be coded on
5 bits.
To allow flexibility and evolution if new codes are introduced, a static
mapping table is associated to each instance of this function.

 {{Fig-code-mapping}} gives a possible mapping, it can be changed to add new codes or reduced if
some values are never used by both ends.

~~~~
            +------+------------------------------+-----------+
            | Code | Description                  | Mapping   |
            +------+------------------------------+-----------+
            | 0.00 |                              |  0x00     |
            | 0.01 | GET                          |  0x01     |
            | 0.02 | POST                         |  0x02     |
            | 0.03 | PUT                          |  0x03     |
            | 0.04 | DELETE                       |  0x04     |
            | 0.05 | FETCH                        |  0x05     |
            | 0.06 | PATCH                        |  0x06     |
            | 0.07 | iPATCH                       |  0x07     |
            | 2.01 | Created                      |  0x08     |
            | 2.02 | Deleted                      |  0x09     |
            | 2.03 | Valid                        |  0x0A     |
            | 2.04 | Changed                      |  0x0B     |
            | 2.05 | Content                      |  0x0C     |
            | 4.00 | Bad Request                  |  0x0D     |
            | 4.01 | Unauthorized                 |  0x0E     |
            | 4.02 | Bad Option                   |  0x0F     |
            | 4.03 | Forbidden                    |  0x10     |
            | 4.04 | Not Found                    |  0x11     |
            | 4.05 | Method Not Allowed           |  0x12     |
            | 4.06 | Not Acceptable               |  0x13     |
            | 4.12 | Precondition Failed          |  0x14     |
            | 4.13 | Request Entity Too Large     |  0x15     |
            | 4.15 | Unsupported Content-Format   |  0x16     |
            | 5.00 | Internal Server Error        |  0x17     |
            | 5.01 | Not Implemented              |  0x18     |
            | 5.02 | Bad Gateway                  |  0x19     |
            | 5.03 | Service Unavailable          |  0x1A     |
            | 5.04 | Gateway Timeout              |  0x1B     |
            | 5.05 | Proxying Not Supported       |  0x1C     |
            +------+------------------------------+-----------+


~~~~
{: #Fig-code-mapping title='CoAP code mapping'}
This CDF can also be applied to path to send a reference instead of the path
value.


#### Remapping

With dynamic mapping, the mapping is done dynamically, which means that the
other end has no way to the learn the original value. This function is not
conservative. The mapping context must be stored in a reliable way on the
compressor, if lost the session with LPWAN node will be lost, which can generate
a traffic increase on the LPWA network.

This function converts a large number to a smaller one and maintain bi-directional
mapping.
If the field has no semantic, such as a CoAP token or a message ID, this
will reduce the
size of the information sent on the link. This mapping only applies for request
compression, answers must keep the value original value.

For instance a compression receives a CoAP request with a large token. The
compressor reduces the token size by allocating a unused value in a smaller
space. When the response come back, the compressor exchange the smallest
token with the original one.

This mean that the compressor must be aware of the CoAP state machine, to
identify a request and its associated response, but also determine when a
token value can be reused.


#### Reduce-entropy

Reduce-entropy is a non-conservative function. the goal is to minimize the
increase in a field value. It may be used for the observe option, all increase
in the original sequence number will lead to an increase of 1 in the compressed
value.

For instance a LPWAN node is a CoAP server and receives Observe responses
coming from an outside client. The client uses a clock to generate Observe
sequence number. If that value has non particular meaning for the CoAP server,
increase of 1 will not change the protocol behavior. Reordering works the
same way as for original Observe.



### CoAP mandatory header

 {{Fig--possible-function}} proposes some function assignments to the CoAP header fields.

~~~~
/--------------------+---------------------+----------------------------------------\
| Field              |Function             | Behavior                               |
+--------------------+---------------------+----------------------------------------+
|version             |not-sent             |version is always the same              |
+--------------------+---------------------+----------------------------------------+
|type                |value-sent           |if all the types are used               |
|                    |static-mapping       |to reduce to one bit if 2 type are used |
|                    |not-sent             |if only one type is used (e.g. NON)     |
+--------------------+---------------------+----------------------------------------+
|token length        |not-sent             |no tokens or fixed size                 |
|                    |compute-token-length |if token size is reduced                |
|                    |value-sent           |token is sent integrally                |
+--------------------+---------------------+----------------------------------------+
|code                |value-sent           |no modification                         |
|                    |static-mapping       |code size reduction                     |
+--------------------+---------------------+----------------------------------------+
|message id          |value-sent           |no modification                         |
|token               |remapping            |reduces message id size                 |
+====================+=====================+========================================+
|Content-Format      |value-sent           |no modification                         |
|Accept              |not-sent             |defined in the rule                     |
|Max-Age             |static-mapping       |map the possible value                  |
+--------------------+---------------------+----------------------------------------+
|Path:               |value-sent           |no modification                         |
|Uri-Host+Uri-Port+  |not-sent             |defined in the rule                     |
|Uri-Path*+Uri-Query*|static-mapping       |a value to define a path                |
|                    |                     |                                        |
|Proxy-Uri           |                     |Note: only the full path is stored in   |
|Proxy-Scheme        |                     |context                                 |
+--------------------+---------------------+----------------------------------------+
|ETag                |value-sent           |Always sent                             |
|Location-Path       |                     |                                        |
|Location-Query      |                     |                                        |
|If-Match            |                     |                                        |
|If-None-Match       |                     |                                        |
|Size1               |                     |                                        |
+--------------------+---------------------+----------------------------------------+

~~~~
{: #Fig--possible-function title="SCHC functions' example assignment for CoAP"}



### Examples of CoAP header compression

#### Mandatory header with CON message

In this first scenario, the LPWAN compressor receives from outside client
a POST message, which is immediately acknowledged by the ES. For this simple
scenario, the rules are described {{Fig-CoAP-header-1}}

~~~~
 rule id 1
+-------------+-------+-----+---------------+----------------+
| Field       |TV     |MO   |CDF            | Sent           |
+=============+=======+=====+===============+================+
|CoAP version | 01    |=    |not-sent       |                |
|CoAP Type    |       |     |value-sent     |TT              |
|CoAP TKL     | 0000  |=    |not-sent       |                |
|CoAP Code    |       |     |static-map     |  CC CCC        |
|CoAP MID     |       |     |dynamic-map    |         M-ID   |
|CoAP Path    |/path  |     |not-sent       |                |
+-------------+-------+-----+---------------+----------------+
~~~~
{: #Fig-CoAP-header-1 title='CoAP Context to compress header without token'}
 {{Fig-CoAP-header-1}} gives a simple compression rule for CoAP headers
without tokens.

The version fields and Token Length are elided. Code has shrunk to 5 bits
using the
static-mapping function. Message-ID has shrunk to 9 bits to preserve alignment
on byte boundary.

 {{Fig-CoAP-3}} shows the time diagram of the exchange. A LPWAN Application Server sends
a CON message. Compression reduces the header sending only the Type, a mapped
code and the Message ID is change to a value on 9 bits. The receiver decompress
the header. The message ID value is changed.

The CON message is a request, therefore the LC process to a dynamic mapping.
When the ES receives the ACK message, this will not initiate locally a the
message ID mapping since it is a response. The LC receives the ACK and uncompress
it to restore the original value. Dynamic Mapping context lifetime follows
the same rules as message ID duration.

~~~~
                   End System                      LPWA LC
                        |                            |
                        |        rule id=1           |<----------------------
                        |<---------------------------|   +-+-+--+----+--------+
  <-------------------- |  TTCC CCCM MMMM MMMM       |   |1|0| 4|0.01| 0x1234 |
 +-+-+--+----+--------+ |  0000 0010 0000 0001       |   |  0xb4   p    a   t |
 |1|0| 1|0.01| 0x0001 | |                            |   |  h   |
 |  0xb4   p    a   t | |                            |   +------+
 |  h   |               |                            |    dynamic mapping
 +------+               |                            |    +--------+--------+
                        |                            |    |0x1234  |  0x01  |
                        |                            |    +--------+--------+
----------------------->|       rule id=1            |
+-+-+--+----+--------+  |--------------------------->|
|1|2| 0|2.05| 0x0001 |  |    TTCC CCCM MMMM MMMM     |------------------------>
+-+-+--+----+--------+  |    1000 0000 0000 0001     | +-+-+--+----+--------+
                        |                            | |1|2| 0|2.05| 0x1234 |
                        v                            v +-+-+--+----+--------+

~~~~
{: #Fig-CoAP-3 title='Compression with global addresses'}
Note that the compressor and decompressor must understand the CoAP protocol:

* The LC compressor detects a new transport request and allocate a new dynamic
  mapping value.

* When receiving a response the ES compressor ES detects that this is a response
  (type=2) therefore the message ID value in unchanged.

* The upstream compressor detects that is an REST answer (code 2.05) therefore
  the path option is not inserted in the uncompress header



#### Exchange with token

The following scenario introduces tokens. The LC manages two remapping contexts.
One for Message ID and the other for token. ES manages one context for Message
ID. Mapping is trigged by the reception of CON messages to compress or CoAP
requests to compress. Note that the compressed message ID size has been reduced
to 7 bits, compared to the previous example, to maintain byte boundary alignment.

~~~~
  +----------------+------------------------+----------------+-----------------+
  | Field          |   Function             | Ctxt Value     | Sent compressed |
  +----------------+------------------------+----------------+-----------------+
  |CoAP version    | not-sent               |                |                 |
  |CoAP Type       | value-sent             |                |TT               |
  |CoAP TKL        | compute-token-length   |                |  LL             |
  |CoAP Code       | map-code               | mapping table  |      CCCC C     |
  |CoAP MID        | remapping              | 7 bits         |            M-ID |
  |CoAP Token      | remapping              | 8 bits         |            token|
  |CoAP Path       | not-sent               |/data/humidity  |
  +----------------+------------------------+----------------+-----------------+
~~~~
{: #Fig-CoAP-header-2 title='CoAP Context to compress header with token'}


~~~~
                   End System                      LPWA LC
                        |                            |
                        |        SHIM=1              |<----------------------
                        |<---------------------------|   +-+-+--+----+--------+
  <-------------------- |  TT LL CCCC  C  MMMMMMM    |   |1|0| 4|0.01| 0x1234 |
 +-+-+--+----+--------+ |  00 01 0000  1  0000001    |   |      DEADBEEF      |
 |1|0| 1|0.01| 0x0001 | |  0000  0001                |   |  0xb4   d    a   t |
 |  01   0xb4   d   a | |  Token                     |   |   a     0x08 h   u |
 |    t   a    0x08 h | |                            |   |   m     i    d   i |
 |    u   m     i   d | |                            |   |   t     y  |
 |    i   t     y  |    |                            |   +------------+
 +-----------------+    |                            | Mid mapping: 1234 -> 1
                        |                            | Tk  mapping: DEADBEEF -> 1
----------------------->|       SHIM=1               |
+-+-+--+----+--------+  |--------------------------->|
|1|2| 0|0.00| 0x0001 |  |    TT LL CCCC C MMMMMMMM   |------------------------>
+-+-+--+----+--------+  |    10 01 0000 0 00000001   | +-+-+--+----+--------+
                        |                            | |1|2| 0|0.00| 0x1234 |
                        |                            | +-+-+--+----+--------+
----------------------->|                            |
+-+-+--+----+--------+  |--------------------------->|
|1|0| 0|2.05| 0xCAFE |  |    TT LL CCCC C MMMMMMMM   |------------------------>
| 0x01    2     5    |  |    00 01 1100 0 00000002   | +-+-+--+----+--------+
+--------------------+  |    0000  0001              | |1|0| 4|2.05| 0x0001 |
                        |    2 5                     | |       DEADBEEF     |
                        |                            | |    2    5 |
Mid mapping: CAFE -> 1  |                            | +-----------+
                        |                            |
                        |                            |<------------------------
                        |<---------------------------| +-+-+--+----+--------+
<-----------------------|     TT LL CCCC C  MMMMMMMM | |1|2| 0|0.00|0x0001  |
+-+-+--+----+--------+  |     10 00 0000 0  00000002 | +-+-+--+----+--------+
|1|2| 0|0.00| 0xCAFE |  |                            |
+-+-+--+----+--------+  |                            |
                        v                            v


~~~~
{: #Fig-CoAP-2 title='Compression with token'}






--- back
